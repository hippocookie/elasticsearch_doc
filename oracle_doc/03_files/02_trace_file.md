## Trace File
Provide tunning information
> DBMS_MONITOR.SESSION_TRACE_ENABLE

- v$view: include tunning information
- audit: record events command
- DBMS_RESOURCE_MANAGER: control cpu, memory resource
- oracle event: create trace info
- DBMS_TRACE: record store procedure call trace, exception, error
- event trigger: record abnormal condition
- SQL_TRACE/DBMS_MONITOR: check execute sql

### create by user
```sql
> alter session set sql_trace = TRUE;
```

#### location
```sql
> show parameter dump_dest;
```
|NAME|TYPE|VALUE|
|---|---|---|
|background_dump_dest|string|/u01/app/oracle/diag/rdbms/xe/XE/trace|
|core_dump_dest|string|/u01/app/oracle/diag/rdbms/xe/XE/cdump|
|user_dump_dest|string|/u01/app/oracle/diag/rdbms/xe/XE/trace|

```sql
> select name, value from v$parameter where name like '%dump_dest%';
```

### Automatic Diagnostic Repository
```sql
> select * from v$diag_info;
```
|NAME|VALUE|
|---|---|
|Diag Enabled|TRUE|
|ADR Base|/u01/app/oracle|
|ADR Home|/u01/app/oracle/diag/rdbms/xe/XE|
|Diag Trace|/u01/app/oracle/diag/rdbms/xe/XE/trace|
|Diag Alert|/u01/app/oracle/diag/rdbms/xe/XE/alert|
|Diag Incident|/u01/app/oracle/diag/rdbms/xe/XE/incident|
|Diag Cdump|/u01/app/oracle/diag/rdbms/xe/XE/cdump|
|Health Monitor|/u01/app/oracle/diag/rdbms/xe/XE/hm|
|Default Trace File|/u01/app/oracle/diag/rdbms/xe/XE/trace/XE_ora_346.trc|
|Active Problem Count|0|
|Active Incident Count|0|

- Diag Trace: trace file location
- Default Trace File: current session trace file

#### mark trace file
```sql
> alter session set tracefile_identifier = 'FIND_ME';
```
Format
- trace file(.trc)
- trace map file(.trm): include trace file structure





