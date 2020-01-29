![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Automatically Create SQL Instance Name For Unattended Config File
**Post Date: November 9, 2016**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>If you're looking to create an unattended installation process, and want to concatenate your own parameters into the configuration file; then here is one way you can do it.

There are a number of assumptions that are made for this process, and there are some generic parameters supplied, but I've run several tests around other configurations and this logic is in perfect working order. I've created several instances automatically.

This is obviously designed for multi-sql server environments where there are many instances under one server.

In this example the names instances have the following framework, and the logic below ensures this convention will be maintained for each additional instance that is created.

SQLSHARE01, …02, …03 etc.

Here's the logic.</p>      


## SQL-Logic
```SQL
use master;
set nocount on
 
declare @server_name    varchar(255) = (select @@servername)
declare @sql_service    varchar(255) = 'MyDomain\MySQLServiceAccount'
declare @sql_service_pw varchar(255) = 'MyPassword'
declare @sql_agent  varchar(255) = 'MyDomain\MyAgentServiceAccount'
declare @sql_agent_pw   varchar(255) = 'MyPassword'
declare @dba_group  varchar(255) = 'MyDomain\MyDBAGroup'
declare @sql_sa     varchar(255) = 'MysaAccount'
declare @sql_sa_pw  varchar(255) = 'MysaPassword'
declare @my_user_data   varchar(255) = 'MyUserDataPath'
declare @my_user_log    varchar(255) = 'MyUserLogPath'
declare @my_temp_data   varchar(255) = 'MyTempDataPath'
declare @my_temp_log    varchar(255) = 'MyTempLogPath'
declare @my_backup_path varchar(255) = 'MyBackupPath'
declare @run_bcp    varchar(8000) 
declare @silent_string  varchar(max)
 
-- create table to hold instance names
declare @sql_instances table
(
    [id]        int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread
insert into @sql_instances ([rootkey], [sql_instances], [value])
execute xp_regread
    @rootkey    = 'hkey_local_machine'
,   @key        = 'software\microsoft\microsoft sql server'
,   @value_name = 'installedinstances'
  
-- set prefix name for multi-instance environment aka: Shared SQL Environment "SQLSHARE"
-- produce the next instance name.
declare @prefix     varchar(255) = 'SQLSHARE'
declare @next_instance  varchar(255) = 
(
select
    case
    when max([sql_instances]) is null then 'SQLSHARE01'
    when max([sql_instances]) = 'MSSQLSERVER' then 'SQLSHARE01'
    else @prefix + cast(cast(right(max([sql_instances]), 2) as int) + 1 as varchar)
    end as [sql_instances]
from
    @sql_instances si
where
    si.[id] > 0
)
-- reference: https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
 
set @silent_string          =
'[OPTIONS]
ACTION              ="Install"
ENU             ="True"
QUIET               ="False"
QUIETSIMPLE         ="True"
UpdateEnabled           ="False"
ERRORREPORTING          ="False"
USEMICROSOFTUPDATE      ="False"
FEATURES            =SQLENGINE,RS
UpdateSource            ="MU"
HELP                ="False"
INDICATEPROGRESS        ="False"
X86             ="False"
INSTALLSHAREDDIR        ="E:\Program Files\Microsoft SQL Server"
INSTALLSHAREDWOWDIR     ="E:\Program Files (x86)\Microsoft SQL Server"
INSTANCENAME            ="' + @next_instance + '"
SQMREPORTING            ="False"
INSTANCEID          ="' + @next_instance + '"
RSINSTALLMODE           ="FilesOnlyMode"
INSTANCEDIR         ="E:\Program Files\Microsoft SQL Server"
AGTSVCACCOUNT           ="MyDomain\MyServiceAccount"
AGTSVCPASSWORD          ="MyPassword"
AGTSVCSTARTUPTYPE       ="Automatic"
COMMFABRICPORT          ="0"
COMMFABRICNETWORKLEVEL          ="0"
COMMFABRICENCRYPTION            ="0"
MATRIXCMBRICKCOMMPORT           ="0"
SQLSVCSTARTUPTYPE       ="Automatic"
FILESTREAMLEVEL         ="0"
ENABLERANU          ="False"
SQLCOLLATION            ="SQL_Latin1_General_CP1_CI_AS"
SQLSVCACCOUNT           ="MyDomain\MyServiceAccount"
SQLSVCPASSWORD          ="MyPassword"
AGTSVCACCOUNT           ="MyDomain\MyServiceAccount"
AGTSVCPASSWORD          ="MyPassword"
SQLSYSADMINACCOUNTS     ="MyDomain\MyServiceAccount" "MyDomain\MyDBAGroup"
SECURITYMODE            ="SQL"
SAPWD               ="##MysaPassword' + right(@next_instance, 2) + '!!##"
SQLTEMPDBDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLTEMPDBLOGDIR         ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLUSERDBDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLUSERDBLOGDIR         ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLBACKUPDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Backup"
ADDCURRENTUSERASSQLADMIN        ="False"
TCPENABLED          ="1"
NPENABLED           ="0"
BROWSERSVCSTARTUPTYPE           ="Automatic"
RSSVCACCOUNT            ="MyDomain\MyServiceAccount"
RSSVCPASSWORD           ="MyPassword"
RSSVCSTARTUPTYPE        ="Automatic"'
 
if object_id('tempdb..##silent_string') is not null drop table ##silent_string
create table ##silent_string ([bcpcommand]  varchar(8000)) insert into  ##silent_string select @silent_string
 
select  @run_bcp = 
'bcp "select [bcpcommand] from ##silent_string" ' + 'queryout "e:\sql_silent_install_config_file\sql_2014_config_file.ini" -c -t -T -S' + @server_name 
exec master..xp_cmdshell @run_bcp
```

Ok; so after you create the configuration file; you'll naturally be compelled to check it out. Tabs, and spacing does not always look so sharp whenever something is exported so when you see it; it's going to look butchered. Tabs, and spaces all a mess, but no worries it's only there so the OS can pick it up, and run through the install. As long as that part works you'll be fine. Each time this is run; it will recreate the former .ini file so no matter what happens to the file after the fact; it will be reproduced.
Of course; whenever you're ready to run this, simply open Command Prompt (or incorporate this into a Job step), and run the following script. Make sure to supply the appropriate path where your installation medium exists.

E:DEPENDENCIESSQL_SERVER_2014STANDARD_X64setup.exe /configurationfile=E:SQL_SILENT_INSTALL_CONFIG_FILEsql_2014_config_file.ini /IacceptSQLServerLicenseTerms

Don't forget people… Standard Edition supports about 16 Instances, while Enterprise can support up to 50. Wouldn't be too hard to add some logic to check on the versioning and simply stop at 16, or 50 depending on the edition, licensing etc.
Here are the instances that were created with this approach.

![SQL Configuration Manager]( https://mikesdatawork.files.wordpress.com/2016/11/image0016.png "SQL Services")
 


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

 
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

