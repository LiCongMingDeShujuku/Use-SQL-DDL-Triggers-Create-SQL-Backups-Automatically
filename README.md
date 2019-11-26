![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用SQLDDL触发器自动创建备份代理作业
#### Use SQL DDL Triggers Create SQL Backups Automatically
**TIME STAMP**

![#](images/##############?raw=true "#")

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
这里有一些DDL触发器，我已经习惯为每个添加的新数据库自动创建备份作业
并在删除数据库时自动删除备份作业。

这个脚本取自一些我的旧笔记，这意味着肯定还有改进的余地。
每次我喜欢把它们修改一下用于其他目的。
例如，假设有人重命名数据库，你可能想要为“ALTER DATABASE”事件合并一个触发器。
因此可以捕获新名称并将其应用于相应的作业。或者你可以通过修改触发器来完全避免这种情况。这时候你要使用数据库ID而不是数据库名称。
使用这些触发器时，你可能会遇到一些技术挑战。例如他们可能不允许你删除备份作业而不先删除数据库。这没什么，只是一起禁用或删除触发器的限制。你可以继续做你要做的事情，或者修改它们。如果能在这个帖子中回复就更好了。

这是一个快速下降声明：


## English
here’s some DDL triggers that i’ve used to automatically create backup jobs for each new database that is added
and will automatically delete the backup job when a database is dropped.

DDL_Trig_AutoCreate_BUJob
DDL_Trig_AutoDelete_BUJob
keep in mind this script is taken from some old notes which means there is certainly room for improvement.
every time i used these i tend to modify it here and there for other purposes.
for example; suppose someone renames a database… well you might want to incorporate a trigger for
‘ALTER DATABASE’ events so the new name can be captured and applied to it’s corresponding job, or you can
avoid this completely by modifying the trigger to use the database id, and not the database name. it’s up to you.
you may find some technical challenges when using these triggers. for example they may not permit you to
drop a backup job without first dropping the database. no bigy; just disable or drop the triggers all together,
and proceed with whatever you were going to do, or better yet… mod them, and post back in this thread 


here’s a quick drop statement:

---
## Logic
```SQL
/************************************/
drop trigger DDL_Trig_AutoDeleteBUJob
on all server
go
/************************************/
 
at the very least these triggers can get you started in creating your own cool DDL automations.  if so... please post
the cool backup/maintenance stuff you've done in this thread to contribute back to the community 
至少这些触发器可以帮助你开始创建自己的DDL自动化。如果是的话，请发帖
你在此主题中所做的很酷的备份/维护工作来帮助大家。
 
ok...  on with the scripts.   here you go.
好的，开始运行脚本
 
 
 
/**************************************************/ /**************************************************/ 
-- This will create a standard SQL Backup Job for all Databases.
-- 这将为所有数据库创建标准SQL备份作业。 /**************************************************/ 
 
declare @mydb varchar(60) 
declare GetDBname cursor read_only for select name from master..sysdatabases 
open GetDBname fetch next from GetDBname into @mydb 
 
declare @desc varchar(50) while @@fetch_status = 0 
begin set @desc = 'Full Database Backup ' + @MYDB exec msdb.dbo.sp_add_job @job_name= @desc
, @enabled=1
, @notify_level_eventlog=0
, @notify_level_email=0
, @notify_level_netsend=0
, @notify_level_page=0
, @delete_level=0
, @description=@desc
, @category_name='[uncategorized (local)]'
, @owner_login_name='sa' fetch next from GetDBname into @mydb end close GetDBname deallocate GetDBname 
go 
 
/**************************************************/ 
declare @mydb varchar(60) declare GetDBname cursor read_only for select name from master..sysdatabases 
open GetDBname fetch next from GetDBname into @mydb
  
declare @desc varchar(50) 
declare @command varchar (200) 
--declare @outputfile varchar (50) 
while @@fetch_status = 0 begin set @desc = 'Full Database Backup ' + @MYDB 
set @command = 'BACKUP DATABASE [' + @mydb + '] TO DISK = N''E:\MSSQL\Backup\' + @mydb + '.bak'' WITH INIT , NOUNLOAD , NAME = N''' + @mydb + ' Backup'', NOSKIP , STATS = 10, NOFORMAT' 
--set @outputfile = 'D:\BackupReport\' + @mydb + '.txt' EXEC msdb.dbo.sp_add_jobstep @job_name = @desc
, @step_name=N'Full Backup'
, @step_id=1
, @cmdexec_success_code=0
, @on_success_action=1
, @on_success_step_id=0
, @on_fail_action=2
, @on_fail_step_id=0
, @retry_attempts=0
, @retry_interval=0
, @os_run_priority=0
, @subsystem=N'TSQL'
, @command=@command
, @database_name=@mydb
, --@output_file_name=@outputfile
, @flags=2 fetch next from GetDBname into @mydb end close GetDBname deallocate GetDBname 
go 
 
/**************************************************/ 
declare @mydb varchar (60) 
declare GetDBname cursor read_only for select name from master..sysdatabases 
open GetDBname fetch next from GetDBname into @mydb 
 
declare @desc varchar(50) while @@fetch_status = 0 begin set @desc = 'Full Database Backup ' + @mydb EXEC msdb.dbo.sp_add_jobschedule @job_name = @desc
, @name=N'Backup'
, @enabled=1
, @freq_type=4
, @freq_interval=1
, @freq_subday_type=1
, @freq_subday_interval=0
, @freq_relative_interval=0
, @freq_recurrence_factor=0
, @active_start_date=20050419
, @active_end_date=99991231
, @active_start_time=0
, @active_end_time=235959 EXEC msdb.dbo.sp_add_jobserver @job_name = @desc
, @server_name = N'(local)' fetch next from GetDBname into @mydb end close GetDBname deallocate GetDBname 
go 
 
/****************************************************************************/ /****************************************************************************/ 
-- this trigger will create a new sql database backup job for each new -- database that is added. /****************************************************************************/
 --drop trigger DDL_Trig_AutoCreateBUJob 
--on all server 
--go 
 
Create trigger DDL_Trig_AutoCreateBUJob on all server for create_database as 
-- print 'create database issued.' 
set nocount on 
declare @mydb varchar(150) declare @desc varchar (150) set @mydb = CAST(eventdata().query('/EVENT_INSTANCE/DatabaseName[1]/text()') as NVarchar(128)) 
set @desc = 'Full Database Backup ' + @MYDB exec msdb.dbo.sp_add_job @job_name= @desc
, @enabled=1
, @notify_level_eventlog=0
, @notify_level_email=0
, @notify_level_netsend=0
, @notify_level_page=0
, @delete_level=0
, @description=@desc
, @category_name='[uncategorized (local)]'
, @owner_login_name='sa' 
 
/**************************************************/ 
declare @command varchar(255) 
--declare @outputfile varchar (50) 
set @command = 'BACKUP DATABASE [' + @mydb + '] TO DISK = N''E:\MSSQL\Backup\' + @mydb + '.bak'' WITH DESCRIPTION = N''Backup occurs once a day.'', NOFORMAT, NOINIT, NAME = N''' + @mydb + ' Full Database Backup'', SKIP, NOREWIND, NOUNLOAD, STATS = 10' 
--set @outputfile = 'D:\BackupReport\' + @mydb + '.txt' EXEC msdb.dbo.sp_add_jobstep @job_name = @desc
, @step_name=N'Full Backup'
, @step_id=1
, @cmdexec_success_code=0
, @on_success_action=1
, @on_success_step_id=0
, @on_fail_action=2
, @on_fail_step_id=0
, @retry_attempts=0
, @retry_interval=0
, @os_run_priority=0
, @subsystem=N'TSQL'
, @command=@command
, @database_name=@mydb
 --@output_file_name=@outputfile, @flags=2 
go
 
 
/****************************************************************************/ /****************************************************************************/ 
-- this trigger will remove the SQL Database Backup job for each database -- that is dropped. 

-- 此触发器将删除每个数据库的SQL数据库备份作业。
/****************************************************************************/ 
--drop trigger DDL_Trig_AutoDeleteBUJob 
--on all server 
--go 
 
Create trigger DDL_Trig_AutoDeleteBUJob on all server for drop_database as 
-- print 'create database issued.' set nocount on declare @mydb varchar(150) declare @desc varchar (150) set @mydb = CAST(eventdata().query('/EVENT_INSTANCE/DatabaseName[1]/text()') as NVarchar(128)) set @desc = 'Full Database Backup ' + @MYDB exec msdb.dbo.sp_delete_job @job_name = @desc


```

希望可以帮到你。(Hope this is helpful.)


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

