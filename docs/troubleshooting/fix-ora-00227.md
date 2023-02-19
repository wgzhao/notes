---
title: 修复 Oracle ORA-00227 错误
description: 这篇摘要描述了如何修复 Oracle 数据库控制文件块损坏(ORA-00227) 的故障
tags: ["oracle", "controlfile", "ora-00227"]
---

# 修复 Oracle ORA-00227 错误

[Source](http://blog.itpub.net/26736162/viewspace-2652072/ "Permalink to ORA-00227: corrupt block detected in control file: (block 16, # blocks 1)")

具体报错信息为：

```
 ORA-00227: corrupt block detected in control file: (block 16, # blocks 1)
```

解决的办法为：

```sql
[oracle@OCPLHR dbs]$ sas
SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 29 14:57:47 2019
Copyright (c) 1982, 2011, Oracle.  All rights reserved.
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
SYS@OCPLHR1> startup force 
ORACLE instance started.
Total System Global Area  626327552 bytes
Fixed Size                  2230952 bytes
Variable Size             553649496 bytes
Database Buffers           62914560 bytes
Redo Buffers                7532544 bytes
ORA-00227: corrupt block detected in control file: (block 16, # blocks 1)
ORA-00202: control file: '/u01/app/oracle/oradata/OCPLHR1/control01.ctl'
[oracle@OCPLHR dbs]$ 
[oracle@OCPLHR dbs]$ cp snapcf_OCPLHR1.f /u01/app/oracle/oradata/OCPLHR1/control01.ctl
[oracle@OCPLHR dbs]$ sas
SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 29 15:06:26 2019
Copyright (c) 1982, 2011, Oracle.  All rights reserved.
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
SYS@OCPLHR1> alter database backup controlfile to trace as '/home/oracle/ctl.txt';
Database altered.
SYS@OCPLHR1> startup force nomount
ORACLE instance started.
Total System Global Area  626327552 bytes
Fixed Size                  2230952 bytes
Variable Size             553649496 bytes
Database Buffers           62914560 bytes
Redo Buffers                7532544 bytes
SYS@OCPLHR1> CREATE CONTROLFILE REUSE DATABASE "OCPLHR1" NORESETLOGS  ARCHIVELOG
      MAXLOGFILES 16
      MAXLOGMEMBERS 3
      MAXDATAFILES 100
      MAXINSTANCES 8
      MAXLOGHISTORY 292
  LOGFILE
    GROUP 1 '/u01/app/oracle/oradata/OCPLHR1/redo01.log'  --SIZE 50M BLOCKSIZE 512,
    GROUP 2 '/u01/app/oracle/oradata/OCPLHR1/redo02.log'  --SIZE 50M BLOCKSIZE 512,
     GROUP 3 '/u01/app/oracle/oradata/OCPLHR1/redo03.log'  --SIZE 50M BLOCKSIZE 512
   -- STANDBY LOGFILE
   DATAFILE
     '/u01/app/oracle/oradata/OCPLHR1/system01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/sysaux01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/undotbs01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/users01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/example01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/a.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/ocplhr1_test01.dbf',
     '/u01/app/oracle/oradata/OCPLHR1/trpdata.dbf'
   CHARACTER SET ZHS16GBK
   ;
Control file created.
SYS@OCPLHR1> 
SYS@OCPLHR1> startup force
ORACLE instance started.
Total System Global Area  626327552 bytes
Fixed Size                  2230952 bytes
Variable Size             553649496 bytes
Database Buffers           62914560 bytes
Redo Buffers                7532544 bytes
Database mounted.
ORA-01113: file 1 needs media recovery
ORA-01110: data file 1: '/u01/app/oracle/oradata/OCPLHR1/system01.dbf'
SYS@OCPLHR1> recover database;
Media recovery complete.
SYS@OCPLHR1> 
SYS@OCPLHR1> alter database open;
alter database open
*
ERROR at line 1:
ORA-00603: ORACLE server session terminated by fatal error
ORA-00600: internal error code, arguments: [2662], [0], [3545903], [0], [3551857], [12583040], [], [], [], [], [], []
ORA-00600: internal error code, arguments: [2662], [0], [3545902], [0], [3551857], [12583040], [], [], [], [], [], []
ORA-01092: ORACLE instance terminated. Disconnection forced
ORA-00600: internal error code, arguments: [2662], [0], [3545900], [0], [3551857], [12583040], [], [], [], [], [], []
Process ID: 14562
Session ID: 96 Serial number: 3
SYS@OCPLHR1> startup force
ORA-24324: service handle not initialized
ORA-01041: internal error. hostdef extension doesn't exist
SYS@OCPLHR1> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@OCPLHR dbs]$ 
[oracle@OCPLHR dbs]$ 
[oracle@OCPLHR dbs]$ sas
SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 29 15:10:44 2019
Copyright (c) 1982, 2011, Oracle.  All rights reserved.
Connected to an idle instance.
SYS@OCPLHR1> startup force
ORACLE instance started.
Total System Global Area  626327552 bytes
Fixed Size                  2230952 bytes
Variable Size             553649496 bytes
Database Buffers           62914560 bytes
Redo Buffers                7532544 bytes
Database mounted.
ORA-01113: file 1 needs media recovery
ORA-01110: data file 1: '/u01/app/oracle/oradata/OCPLHR1/system01.dbf'
SYS@OCPLHR1> recover database;
Media recovery complete.
SYS@OCPLHR1> alter database open;
Database altered.
SYS@OCPLHR1> 
SYS@OCPLHR1> alter system switch logfile;
System altered.
SYS@OCPLHR1> alter system switch logfile;
System altered.
SYS@OCPLHR1> startup force
ORACLE instance started.
Total System Global Area  626327552 bytes
Fixed Size                  2230952 bytes
Variable Size             553649496 bytes
Database Buffers           62914560 bytes
Redo Buffers                7532544 bytes
Database mounted.
Database opened.
SYS@OCPLHR1> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@OCPLHR dbs]$ rman target /
Recovery Manager: Release 11.2.0.3.0 - Production on Mon Jul 29 15:11:55 2019
Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.
connected to target database: OCPLHR1 (DBID=2909198110)
RMAN> backup database;
Starting backup at 2019-07-29 15:12:09
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=129 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00001 name=/u01/app/oracle/oradata/OCPLHR1/system01.dbf
input datafile file number=00002 name=/u01/app/oracle/oradata/OCPLHR1/sysaux01.dbf
input datafile file number=00005 name=/u01/app/oracle/oradata/OCPLHR1/example01.dbf
input datafile file number=00003 name=/u01/app/oracle/oradata/OCPLHR1/undotbs01.dbf
input datafile file number=00004 name=/u01/app/oracle/oradata/OCPLHR1/users01.dbf
input datafile file number=00008 name=/u01/app/oracle/oradata/OCPLHR1/trpdata.dbf
input datafile file number=00006 name=/u01/app/oracle/oradata/OCPLHR1/a.dbf
input datafile file number=00007 name=/u01/app/oracle/oradata/OCPLHR1/ocplhr1_test01.dbf
channel ORA_DISK_1: starting piece 1 at 2019-07-29 15:12:11
channel ORA_DISK_1: finished piece 1 at 2019-07-29 15:13:46
piece handle=/u01/app/oracle/fast_recovery_area/OCPLHR1/backupset/2019_07_29/o1_mf_nnndf_TAG20190729T151210_gmx72cd9_.bkp tag=TAG20190729T151210 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:01:35
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
including current SPFILE in backup set
channel ORA_DISK_1: starting piece 1 at 2019-07-29 15:13:47
channel ORA_DISK_1: finished piece 1 at 2019-07-29 15:13:48
piece handle=/u01/app/oracle/fast_recovery_area/OCPLHR1/backupset/2019_07_29/o1_mf_ncsnf_TAG20190729T151210_gmx75chn_.bkp tag=TAG20190729T151210 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2019-07-29 15:13:48
RMAN> sql 'alter system switch logfile';
sql statement: alter system switch logfile
RMAN> backup archivelog all ;
Starting backup at 2019-07-29 15:15:18
current log archived
using channel ORA_DISK_1
channel ORA_DISK_1: starting archived log backup set
channel ORA_DISK_1: specifying archived log(s) in backup set
input archived log thread=1 sequence=1 RECID=1 STAMP=1014909024
input archived log thread=1 sequence=2 RECID=2 STAMP=1014909024
input archived log thread=1 sequence=3 RECID=3 STAMP=1014909024
input archived log thread=1 sequence=4 RECID=4 STAMP=1014909073
input archived log thread=1 sequence=5 RECID=5 STAMP=1014909090
input archived log thread=1 sequence=6 RECID=6 STAMP=1014909092
input archived log thread=1 sequence=7 RECID=7 STAMP=1014909108
input archived log thread=1 sequence=8 RECID=8 STAMP=1014909304
input archived log thread=1 sequence=9 RECID=9 STAMP=1014909318
channel ORA_DISK_1: starting piece 1 at 2019-07-29 15:15:18
channel ORA_DISK_1: finished piece 1 at 2019-07-29 15:15:19
piece handle=/u01/app/oracle/fast_recovery_area/OCPLHR1/backupset/2019_07_29/o1_mf_annnn_TAG20190729T151518_gmx786b1_.bkp tag=TAG20190729T151518 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2019-07-29 15:15:19
RMAN> backup current controlfile;
Starting backup at 2019-07-29 15:15:36
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
channel ORA_DISK_1: starting piece 1 at 2019-07-29 15:15:37
channel ORA_DISK_1: finished piece 1 at 2019-07-29 15:15:38
piece handle=/u01/app/oracle/fast_recovery_area/OCPLHR1/backupset/2019_07_29/o1_mf_ncnnf_TAG20190729T151536_gmx78swd_.bkp tag=TAG20190729T151536 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 2019-07-29 15:15:38
RMAN> exit
Recovery Manager complete.
[oracle@OCPLHR dbs]$ sas
SQL*Plus: Release 11.2.0.3.0 Production on Mon Jul 29 15:15:44 2019
Copyright (c) 1982, 2011, Oracle.  All rights reserved.
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
SYS@OCPLHR1> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     8
Next log sequence to archive   10
Current log sequence           10
SYS@OCPLHR1>
```