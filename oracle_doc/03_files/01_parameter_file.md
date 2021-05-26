## Parameter File
### Type
1. init file: init{oracle_sid}.ora
  - plan text
2. server parameter file: spfile{oracle_sid}.ora
  - binary

check value
```sql
select value from v$parameter where name = 'db_block_size'
/
show parameter db_block_s
```

### init.ora parameter file(PFile)
This file include simple key-value pair, and at least contain two things: database name and the location of contorl file, which is used for finding the location of all other files.

#### location
> $ORACLE_HOME/dbs # linux
> %ORACLE_HOME%\DATABASE  # windows

ALTER SYSTEM command won't write changes into init.ora, if you want to persist changes, you have to modify this file manually.

### server paramter file(SPFile)
It store in database server, not client side.
It solved:
1. Multi client version of paramter file
2. Can't modify via text editor, use ALTER SYSTEM command

#### SPFile to PFile
```sql
> show paramter pfile;
NAME   TYPE   VALUE                                              
------ ------ -------------------------------------------------- 
spfile string /u01/app/oracle/product/11.2.0/xe/dbs/spfileXE.ora

> create spfile from pfile;
```

#### set SPFile
Using ALTER SYSTEM SET command to update SPFile

```sql
Alter system set paramter=value <commont='text'> <defered> <scope=memory|spfile|both> <sid='sid|*'> <container=current|all>
```
- comment: set up comment and will display on v$parameter update_comment column
- defered: affect later session
```sql
select name, issys_modifiable from v$parameter where issys_modifiable = 'DEFERRED';
```
- scope: memory(only this instance, cancelled after restart), spfile(write to spfile, affect after restart), both(default)
- sid: for single instance in cluster
- container: current|all, apply to current session or all plugable database

### reset SPFile
Cancel parameter value, use default
```sql
alter system reset sort_area_size scope=spfile;
```




