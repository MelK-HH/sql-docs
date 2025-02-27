---
title: Deploy a split-merge service
description: Use the split-merge too to move data between sharded databases.
author: scoriani
ms.author: scoriani
ms.reviewer: wiassaf, mathoma
ms.date: 12/04/2018
ms.service: sql-database
ms.subservice: scale-out
ms.topic: how-to
ms.custom: "sqldbrb=1"
---
# Deploy a split-merge service to move data between sharded databases
[!INCLUDE[appliesto-sqldb](../includes/appliesto-sqldb.md)]

The split-merge tool lets you move data between sharded databases. See [Moving data between scaled-out cloud databases](elastic-scale-overview-split-and-merge.md)

## Download the Split-Merge packages

1. Download the latest NuGet version from [NuGet](https://docs.nuget.org/docs/start-here/installing-nuget).

1. Open a command prompt and navigate to the directory where you downloaded nuget.exe. The download includes PowerShell commands.

1. Download the latest Split-Merge package into the current directory with the below command:

   ```cmd
   nuget install Microsoft.Azure.SqlDatabase.ElasticScale.Service.SplitMerge
   ```  

The files are placed in a directory named **Microsoft.Azure.SqlDatabase.ElasticScale.Service.SplitMerge.x.x.xxx.x** where *x.x.xxx.x* reflects the version number. Find the split-merge Service files in the **content\splitmerge\service** sub-directory, and the Split-Merge PowerShell scripts (and required client dlls) in the **content\splitmerge\powershell** sub-directory.

## Prerequisites

1. Create an Azure SQL Database database that will be used as the split-merge status database. Go to the [Azure portal](https://portal.azure.com). Create a new **SQL Database**. Give the database a name and create a new administrator and password. Be sure to record the name and password for later use.

1. Ensure that your server allows Azure Services to connect to it. In the portal, in the **Firewall Settings**, ensure the **Allow access to Azure Services** setting is set to **On**. Click the "save" icon.

1. Create an Azure Storage account for diagnostics output.

1. Create an Azure Cloud Service for your Split-Merge service.

## Configure your Split-Merge service

### Split-Merge service configuration

1. In the folder into which you downloaded the Split-Merge assemblies, create a copy of the *ServiceConfiguration.Template.cscfg* file that shipped alongside *SplitMergeService.cspkg* and rename it *ServiceConfiguration.cscfg*.

1. Open *ServiceConfiguration.cscfg* in a text editor such as Visual Studio that validates inputs such as the format of certificate thumbprints.

1. Create a new database or choose an existing database to serve as the status database for Split-Merge operations and retrieve the connection string of that database.

   > [!IMPORTANT]
   > At this time, the status database must use the Latin  collation (SQL\_Latin1\_General\_CP1\_CI\_AS). For more information, see [Windows Collation Name (Transact-SQL)](/sql/t-sql/statements/windows-collation-name-transact-sql).

   With Azure SQL Database, the connection string typically is of the form:

      `Server=<serverName>.database.windows.net; Database=<databaseName>;User ID=<userId>; Password=<password>; Encrypt=True; Connection Timeout=30`

1. Enter this connection string in the *.cscfg* file in both the **SplitMergeWeb** and **SplitMergeWorker** role sections in the ElasticScaleMetadata setting.

1. For the **SplitMergeWorker** role, enter a valid connection string to Azure storage for the **WorkerRoleSynchronizationStorageAccountConnectionString** setting.

### Configure security

For detailed instructions to configure the security of the service, refer to the [Split-Merge security configuration](elastic-scale-split-merge-security-configuration.md).

For the purposes of a simple test deployment for this tutorial, a minimal set of configuration steps will be performed to get the service up and running. These steps enable only the one machine/account executing them to communicate with the service.

### Create a self-signed certificate

Create a new directory and from this directory execute the following command using a [Developer Command Prompt for Visual Studio](/dotnet/framework/tools/developer-command-prompt-for-vs) window:

   ```cmd
   makecert ^
    -n "CN=*.cloudapp.net" ^
    -r -cy end -sky exchange -eku "1.3.6.1.5.5.7.3.1,1.3.6.1.5.5.7.3.2" ^
    -a sha256 -len 2048 ^
    -sr currentuser -ss root ^
    -sv MyCert.pvk MyCert.cer
   ```

You are asked for a password to protect the private key. Enter a strong password and confirm it. You are then prompted for the password to be used once more after that. Click **Yes** at the end to import it to the Trusted Certification Authorities Root store.

### Create a PFX file

Execute the following command from the same window where makecert was executed; use the same password that you used to create the certificate:

   ```cmd
   pvk2pfx -pvk MyCert.pvk -spc MyCert.cer -pfx MyCert.pfx -pi <password>
   ```

### Import the client certificate into the personal store

1. In Windows Explorer, double-click *MyCert.pfx*.
2. In the **Certificate Import Wizard** select **Current User** and click **Next**.
3. Confirm the file path and click **Next**.
4. Type the password, leave **Include all extended properties** checked and click **Next**.
5. Leave **Automatically select the certificate store[…]** checked and click **Next**.
6. Click **Finish** and **OK**.

### Upload the PFX file to the cloud service

1. Go to the [Azure portal](https://portal.azure.com).
2. Select **Cloud Services**.
3. Select the cloud service you created above for the Split/Merge service.
4. Click **Certificates** on the top menu.
5. Click **Upload** in the bottom bar.
6. Select the PFX file and enter the same password as above.
7. Once completed, copy the certificate thumbprint from the new entry in the list.

### Update the service configuration file

Paste the certificate thumbprint copied above into the thumbprint/value attribute of these settings.
For the worker role:

   ```xml
    <Setting name="DataEncryptionPrimaryCertificateThumbprint" value="" />
    <Certificate name="DataEncryptionPrimary" thumbprint="" thumbprintAlgorithm="sha1" />
   ```

For the web role:

   ```xml
    <Setting name="AdditionalTrustedRootCertificationAuthorities" value="" />
    <Setting name="AllowedClientCertificateThumbprints" value="" />
    <Setting name="DataEncryptionPrimaryCertificateThumbprint" value="" />
    <Certificate name="SSL" thumbprint="" thumbprintAlgorithm="sha1" />
    <Certificate name="CA" thumbprint="" thumbprintAlgorithm="sha1" />
    <Certificate name="DataEncryptionPrimary" thumbprint="" thumbprintAlgorithm="sha1" />
   ```

Please note that for production deployments separate certificates should be used for the CA, for encryption, the Server certificate and client certificates. For detailed instructions on this, see [Security Configuration](elastic-scale-split-merge-security-configuration.md).

## Deploy your service

1. Go to the [Azure portal](https://portal.azure.com)
2. Select the cloud service that you created earlier.
3. Click **Overview**.
4. Choose the staging environment, then click **Upload**.
5. In the dialog box, enter a deployment label. For both 'Package' and 'Configuration', click 'From Local' and choose the *SplitMergeService.cspkg* file and your cscfg file that you configured earlier.
6. Ensure that the checkbox labeled **Deploy even if one or more roles contain a single instance** is checked.
7. Hit the tick button in the bottom right to begin the deployment. Expect it to take a few minutes to complete.

## Troubleshoot the deployment

If your web role fails to come online, it is likely a problem with the security configuration. Check that the TLS/SSL is configured as described above.

If your worker role fails to come online, but your web role succeeds, it is most likely a problem connecting to the status database that you created earlier.

- Make sure that the connection string in your cscfg is accurate.
- Check that the server and database exist, and that the user id and password are correct.
- For Azure SQL Database, the connection string should be of the form:

   `Server=<serverName>.database.windows.net; Database=<databaseName>;User ID=<user>; Password=<password>; Encrypt=True; Connection Timeout=30`

- Ensure that the server name does not begin with **https://**.
- Ensure that your server allows Azure Services to connect to it. To do this, open your database in the portal and ensure that the **Allow access to Azure Services** setting is set to **On****.

## Test the service deployment

### Connect with a web browser

Determine the web endpoint of your Split-Merge service. You can find this in the portal by going to the **Overview** of your cloud service and looking under **Site URL** on the right side. Replace **http://** with **https://** since the default security settings disable the HTTP endpoint. Load the page for this URL into your browser.

### Test with PowerShell scripts

The deployment and your environment can be tested by running the included sample PowerShell scripts.

> [!IMPORTANT]
> The sample scripts run on PowerShell 5.1. They do not currently run on PowerShell 6 or later.

The script files included are:

1. *SetupSampleSplitMergeEnvironment.ps1* - sets up a test data tier for Split/Merge (see table below for detailed description)
2. *ExecuteSampleSplitMerge.ps1* - executes test operations on the test data tier (see table below for detailed description)
3. *GetMappings.ps1* - top-level sample script that prints out the current state of the shard mappings.
4. *ShardManagement.psm1*  - helper script that wraps the ShardManagement API
5. *SqlDatabaseHelpers.psm1* - helper script for creating and managing databases in SQL Database

   <table width="100%">
     <tr>
       <th>PowerShell file</th>
       <th>Steps</th>
     </tr>
     <tr>
       <th rowspan="5">SetupSampleSplitMergeEnvironment.ps1</th>
       <td>1. Creates a shard map manager database</td>
     </tr>
     <tr>
       <td>2. Creates 2 shard databases.
     </tr>
     <tr>
       <td>3. Creates a shard map for those databases (deletes any existing shard maps on those databases). </td>
     </tr>
     <tr>
       <td>4. Creates a small sample table in both the shards, and populates the table in one of the shards.</td>
     </tr>
     <tr>
       <td>5. Declares the SchemaInfo for the sharded table.</td>
     </tr>
   </table>
   <table width="100%">
     <tr>
       <th>PowerShell file</th>
       <th>Steps</th>
     </tr>
   <tr>
       <th rowspan="4">ExecuteSampleSplitMerge.ps1 </th>
       <td>1. Sends a split request to the Split-Merge Service web frontend, which splits half the data from the first shard to the second shard.</td>
     </tr>
     <tr>
       <td>2. Polls the web frontend for the split request status and waits until the request completes.</td>
     </tr>
     <tr>
       <td>3. Sends a merge request to the Split-Merge Service web frontend, which moves the data from the second shard back to the first shard.</td>
     </tr>
     <tr>
       <td>4. Polls the web frontend for the merge request status and waits until the request completes.</td>
     </tr>
   </table>

## Use PowerShell to verify your deployment

1. Open a new PowerShell window and navigate to the directory where you downloaded the Split-Merge package, and then navigate into the "powershell" directory.

2. Create a server (or choose an existing server) where the shard map manager and shards will be created.

   > [!NOTE]
   > The *SetupSampleSplitMergeEnvironment.ps1* script creates all these databases on the same server by default to keep the script simple. This is not a restriction of the Split-Merge Service itself.

   A SQL authentication login with read/write access to the DBs will be needed for the Split-Merge service to move data and update the shard map. Since the Split-Merge Service runs in the cloud, it does not currently support Integrated Authentication.

   Make sure the server is configured to allow access from the IP address of the machine running these scripts. You can find this setting under SQL server / Firewalls and virtual networks / Client IP addresses.

3. Execute the *SetupSampleSplitMergeEnvironment.ps1* script to create the sample environment.

   Running this script will wipe out any existing shard map management data structures on the shard map manager database and the shards. It may be useful to rerun the script if you wish to re-initialize the shard map or shards.

   Sample command line:

   ```cmd
   .\SetupSampleSplitMergeEnvironment.ps1
    -UserName 'mysqluser' -Password 'MySqlPassw0rd' -ShardMapManagerServerName 'abcdefghij.database.windows.net'
   ```

4. Execute the Getmappings.ps1 script to view the mappings that currently exist in the sample environment.

   ```cmd
   .\GetMappings.ps1
    -UserName 'mysqluser' -Password 'MySqlPassw0rd' -ShardMapManagerServerName 'abcdefghij.database.windows.net'
   ```

5. Execute the *ExecuteSampleSplitMerge.ps1* script to execute a split operation (moving half the data on the first shard to the second shard) and then a merge operation (moving the data back onto the first shard). If you configured TLS and left the http endpoint disabled, ensure that you use the https:// endpoint instead.

    Sample command line:

    ```cmd
    .\ExecuteSampleSplitMerge.ps1
    -UserName 'mysqluser' -Password 'MySqlPassw0rd'
    -ShardMapManagerServerName 'abcdefghij.database.windows.net'
    -SplitMergeServiceEndpoint 'https://mysplitmergeservice.cloudapp.net'
    -CertificateThumbprint '0123456789abcdef0123456789abcdef01234567'
    ```

    If you receive the below error, it is most likely a problem with your Web endpoint's certificate. Try connecting to the Web endpoint with your favorite Web browser and check if there is a certificate error.

    `Invoke-WebRequest : The underlying connection was closed: Could not establish trust relationship for the SSL/TLSsecure channel.`

    If it succeeded, the output should look like the below:

    ```output
    > .\ExecuteSampleSplitMerge.ps1 -UserName 'mysqluser' -Password 'MySqlPassw0rd' -ShardMapManagerServerName 'abcdefghij.database.windows.net' -SplitMergeServiceEndpoint 'http://mysplitmergeservice.cloudapp.net' -CertificateThumbprint 0123456789abcdef0123456789abcdef01234567
    > Sending split request
    > Began split operation with id dc68dfa0-e22b-4823-886a-9bdc903c80f3
    > Polling split-merge request status. Press Ctrl-C to end
    > Progress: 0% | Status: Queued | Details: [Informational] Queued request
    > Progress: 5% | Status: Starting | Details: [Informational] Starting split-merge state machine for request.
    > Progress: 5% | Status: Starting | Details: [Informational] Performing data consistency checks on target     shards.
    > Progress: 20% | Status: CopyingReferenceTables | Details: [Informational] Moving reference tables from     source to target shard.
    > Progress: 20% | Status: CopyingReferenceTables | Details: [Informational] Waiting for reference tables copy     completion.
    > Progress: 20% | Status: CopyingReferenceTables | Details: [Informational] Moving reference tables from     source to target shard.
    > Progress: 44% | Status: CopyingShardedTables | Details: [Informational] Moving key range [100:110) of     Sharded tables
    > Progress: 44% | Status: CopyingShardedTables | Details: [Informational] Successfully copied key range     [100:110) for table [dbo].[MyShardedTable]
    > ...
    > ...
    > Progress: 90% | Status: Completing | Details: [Informational] Successfully deleted shardlets in table     [dbo].[MyShardedTable].
    > Progress: 90% | Status: Completing | Details: [Informational] Deleting any temp tables that were created     while processing the request.
    > Progress: 100% | Status: Succeeded | Details: [Informational] Successfully processed request.
    > Sending merge request
    > Began merge operation with id 6ffc308f-d006-466b-b24e-857242ec5f66
    > Polling request status. Press Ctrl-C to end
    > Progress: 0% | Status: Queued | Details: [Informational] Queued request
    > Progress: 5% | Status: Starting | Details: [Informational] Starting split-merge state machine for request.
    > Progress: 5% | Status: Starting | Details: [Informational] Performing data consistency checks on target     shards.
    > Progress: 20% | Status: CopyingReferenceTables | Details: [Informational] Moving reference tables from     source to target shard.
    > Progress: 44% | Status: CopyingShardedTables | Details: [Informational] Moving key range [100:110) of     Sharded tables
    > Progress: 44% | Status: CopyingShardedTables | Details: [Informational] Successfully copied key range     [100:110) for table [dbo].[MyShardedTable]
    > ...
    > ...
    > Progress: 90% | Status: Completing | Details: [Informational] Successfully deleted shardlets in table     [dbo].[MyShardedTable].
    > Progress: 90% | Status: Completing | Details: [Informational] Deleting any temp tables that were created     while processing the request.
    > Progress: 100% | Status: Succeeded | Details: [Informational] Successfully processed request.
    >
    ```

6. Experiment with other data types! All of these scripts take an optional -ShardKeyType parameter that allows you to specify the key type. The default is Int32, but you can also specify Int64, Guid, or Binary.

## Create requests

The service can be used either by using the web UI or by importing and using the SplitMerge.psm1 PowerShell module which will submit your requests through the web role.

The service can move data in both sharded tables and reference tables. A sharded table has a sharding key column and has different row data on each shard. A reference table is not sharded so it contains the same row data on every shard. Reference tables are useful for data that does not change often and is used to JOIN with sharded tables in queries.

In order to perform a split-merge operation, you must declare the sharded tables and reference tables that you want to have moved. This is accomplished with the **SchemaInfo** API. This API is in the **Microsoft.Azure.SqlDatabase.ElasticScale.ShardManagement.Schema** namespace.

1. For each sharded table, create a **ShardedTableInfo** object describing the table's parent schema name (optional, defaults to "dbo"), the table name, and the column name in that table that contains the sharding key.
2. For each reference table, create a **ReferenceTableInfo** object describing the table's parent schema name (optional, defaults to "dbo") and the table name.
3. Add the above TableInfo objects to a new **SchemaInfo** object.
4. Get a reference to a **ShardMapManager** object, and call **GetSchemaInfoCollection**.
5. Add the **SchemaInfo** to the **SchemaInfoCollection**, providing the shard map name.

An example of this can be seen in the SetupSampleSplitMergeEnvironment.ps1 script.

The Split-Merge service does not create the target database (or schema for any tables in the database) for you. They must be pre-created before sending a request to the service.

## Troubleshooting

You may see the below message when running the sample PowerShell scripts:

   `Invoke-WebRequest : The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel.`

This error means that your TLS/SSL certificate is not configured correctly. Please follow the instructions in section 'Connecting with a web browser'.

If you cannot submit requests you may see this:

   `[Exception] System.Data.SqlClient.SqlException (0x80131904): Could not find stored procedure 'dbo.InsertRequest'.`

In this case, check your configuration file, in particular the setting for **WorkerRoleSynchronizationStorageAccountConnectionString**. This error typically indicates that the worker role could not successfully initialize the metadata database on first use.

[!INCLUDE [elastic-scale-include](../includes/elastic-scale-include.md)]

<!--Image references-->
[1]: ./media/sql-database-elastic-scale-configure-deploy-split-and-merge/allowed-services.png
[2]: ./media/sql-database-elastic-scale-configure-deploy-split-and-merge/manage.png
[3]: ./media/sql-database-elastic-scale-configure-deploy-split-and-merge/staging.png
[4]: ./media/sql-database-elastic-scale-configure-deploy-split-and-merge/upload.png
[5]: ./media/sql-database-elastic-scale-configure-deploy-split-and-merge/storage.png
