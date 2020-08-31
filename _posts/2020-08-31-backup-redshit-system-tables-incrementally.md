---
layout: post
date: 2020-08-31 13:30:00 +0530
title: Backup RedShit System Tables Incrementally
description: Backup or Export the AWS RedShift System tables to other tables. These exports are happening incrementally.
categories:
- RedShift
tags:
- aws
- redshift
- lambda
- python
- github
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/Export RedShift System Tables And Views To S3.jpg"
---
**Export or Backup the AWS RedShift System tables to other tables. These exports are happening incrementally.**

[Last time](https://thedataguy.in/export-redshift-system-tables-views-to-s3/) I have created a stored procedure to export the RedShift's system tables to S3 buckets incrementally. Then we can query them via RedShift Spectrum or AWS Athena. We have to spend some time to setup the Athena or Spectrum. So this time I came with another option that we can export the tables to different tables. So it can be directly accessed inside the Redshift. 

## Create a sperate schema and Tables:

Im going to backup some of the system tables to the schema `systable`.  The list of tables that Im going to export.

* stl_alert_event_log
* stl_ddltext
* stl_explain
* stl_query
* stl_query_metrics
* stl_querytext
* stl_utilitytext
* stl_vacuum
* stl_wlm_query
* svl_query_summary
* svv_table_info

**[Click here](https://github.com/BhuviTheDataGuy/RedShift-ToolKit/blob/master/BackupSysTables/table-ddl.sql)** to get the above table's DDL.

Create a metadata table to maintain the last exported timestamp.
```sql
CREATE TABLE IF not EXISTS systable.history_export 
( 
	id                                 INTEGER identity encode az64 , 
	export_timestamp timestamp without TIME zone DEFAULT current_timestamp encode az64 ,
	object_name                        VARCHAR(100) encode lzo , 
	start_time timestamp without       TIME zone encode az64 , 
	export_query                       VARCHAR(65000) encode lzo 
);
```
## Stored Procedure for Export:

Create the stored procedure to backup the system tables to `systable` schema.
```sql
CREATE OR REPLACE PROCEDURE "systable".backup_systables()
	LANGUAGE plpgsql
AS $$
	
	
DECLARE
-- Store the max Timestamp
stlq_starttime timestamp;
ddlt_starttime timestamp;
utlt_starttime timestamp;
wq_starttime timestamp;
vac_starttime timestamp;


-- Declare queries as variable
stl_query_export varchar(65000);
stl_ddltext_export varchar(65000);
stl_utilitytext_export varchar(65000);
stl_querytext_export varchar(65000);
stl_wlm_query_export varchar(65000);
stl_explain_export varchar(65000);
svl_query_summary_export varchar(65000);
stl_query_metrics_export varchar(65000);
stl_alert_event_log_export varchar(65000);
stl_vacuum_export varchar(65000);
svv_table_info_export varchar(65000);

BEGIN


-- Find the current max timestamp 
select max(starttime) from stl_query into stlq_starttime;
select max(starttime) from stl_ddltext into ddlt_starttime;
select max(starttime) from stl_utilitytext into utlt_starttime;
select max(service_class_start_time) from stl_wlm_query into wq_starttime;
select max(eventtime) from stl_vacuum into vac_starttime;

-- Start exporting the data
RAISE info 'Starting to export stl_query';
stl_query_export='insert into systable.dba_stl_query SELECT stlq.* FROM stl_query  stlq,   
(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime
 FROM systable.history_export where object_name=\'stl_query\') h WHERE stlq.starttime > h.max_starttime';

execute stl_query_export;     
insert into systable.history_export (object_name,start_time,export_query) values ('stl_query',stlq_starttime,stl_query_export);

RAISE info 'Starting to export stl_ddltext';
stl_ddltext_export='insert into systable.dba_stl_ddltext SELECT ddlt.userid,ddlt.xid,ddlt.pid,ddlt.label,ddlt.starttime, ddlt.endtime,ddlt.sequence,replace(replace(ddlt.text,\'\\\\n\',\'\'),\'\\\\r\',\'\') as text FROM stl_ddltext ddlt,   (SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_ddltext\') h WHERE ddlt.starttime > h.max_starttime';

execute stl_ddltext_export;     
insert into systable.history_export (object_name,start_time,export_query) values ('stl_ddltext',ddlt_starttime,stl_ddltext_export);

RAISE info 'Starting to export stl_utilitytext';
stl_utilitytext_export='insert into systable.dba_stl_utilitytext SELECT utlt.* FROM stl_utilitytext utlt, (SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_utilitytext\') h WHERE utlt.starttime > h.max_starttime';

execute stl_utilitytext_export;     
insert into systable.history_export (object_name,start_time,export_query) values ('stl_utilitytext',utlt_starttime,stl_utilitytext_export);

RAISE info 'Starting to export stl_querytext';
stl_querytext_export='insert into systable.dba_stl_querytext select qtxt.userid,qtxt.xid,qtxt.pid,qtxt.query,qtxt.sequence,replace(replace(qtxt.text,\'\\\\n\',\'\'),\'\\\\r\',\'\') as text from stl_querytext qtxt join stl_query qt on qt.userid=qtxt.userid and qt.query=qtxt.query and qt.xid=qtxt.xid and qt.pid=qtxt.pid where qt.starttime>(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_query\')';

execute stl_querytext_export;     
insert into systable.history_export (object_name,export_query) values ('stl_querytext',stl_querytext_export);

RAISE info 'Starting to export stl_wlm_query';
stl_wlm_query_export='insert into systable.dba_stl_wlm_query SELECT wq.* FROM stl_wlm_query wq, (SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_service_class_start_time FROM systable.history_export where object_name=\'stl_wlm_query\') h WHERE wq.service_class_start_time > h.max_service_class_start_time';

execute stl_wlm_query_export;     
insert into systable.history_export (object_name,start_time,export_query) values ('stl_wlm_query',stlq_starttime,stl_wlm_query_export);

RAISE info 'Starting to export stl_explain';
stl_explain_export='insert into systable.dba_stl_explain select exp.* from stl_explain exp join stl_query stlq on stlq.userid=exp.userid and stlq.query=exp.query where stlq.starttime>(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_query\')';

execute stl_explain_export;     
insert into systable.history_export (object_name,export_query) values ('stl_explain',stl_explain_export);

RAISE info 'Starting to export svl_query_summary';
svl_query_summary_export='insert into systable.dba_svl_query_summary select qs.* from svl_query_summary qs join stl_query stlq on stlq.userid=qs.userid and stlq.query=qs.query  where stlq.starttime>(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_query\')';


execute svl_query_summary_export;     
insert into systable.history_export (object_name,export_query) values ('svl_query_summary',svl_query_summary_export);


RAISE info 'Starting to export stl_query_metrics';
stl_query_metrics_export='insert into systable.dba_stl_query_metrics select qm.* from stl_query_metrics qm join stl_query stlq on stlq.userid=qm.userid and stlq.query=qm.query  where stlq.starttime>(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_query\')';

execute stl_query_metrics_export;     
insert into systable.history_export (object_name,export_query) values ('stl_query_metrics',stl_query_metrics_export);

RAISE info 'Starting to export stl_alert_event_log';
stl_alert_event_log_export='insert into systable.dba_stl_alert_event_log select evnt.* from stl_alert_event_log evnt join stl_query qt on qt.userid=evnt.userid and qt.query=evnt.query and qt.xid=evnt.xid and qt.pid=evnt.pid  where qt.starttime>(SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where object_name=\'stl_query\')';

execute stl_alert_event_log_export;     
insert into systable.history_export (object_name,export_query) values ('stl_alert_event_log',stl_alert_event_log_export);

RAISE info 'Starting to export stl_vacuum';
stl_query_export='insert into systable.dba_stl_vacuum SELECT stlv.* FROM stl_vacuum  stlv,   (SELECT NVL(MAX(start_time),\'1902-01-01\'::TIMESTAMP) AS max_starttime FROM systable.history_export where 
object_name=\'stl_vacuum\') h WHERE stlv.eventtime > h.max_starttime';


execute stl_query_export;     
insert into systable.history_export (object_name,start_time,export_query) values ('stl_vacuum',vac_starttime,stl_query_export);

RAISE info 'Starting to export svv_table_info';
svv_table_info_export='insert into systable.dba_svv_table_info select DATE(CURRENT_TIMESTAMP) exportdate,* from svv_table_info';

execute svv_table_info_export;     
insert into systable.history_export (object_name,export_query) values ('svv_table_info',svv_table_info_export);


END
$$
;
```
## Execute the procedure:

```sql
CALL "systable".backup_systables()

INFO: Starting to export stl_query
INFO: Starting to export stl_ddltext
INFO: Starting to export stl_utilitytext
```
### Reference:

If you are interested to export these system tables directly to S3, then please follow his **[blog post](https://thedataguy.in/export-redshift-system-tables-views-to-s3/)**