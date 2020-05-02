---
title: Export RedShift System Tables And Views To S3
date: 2020-04-06 06:45:00 +0530
description: Export the ResShift system tables and views (STL tables) to S3 for persistent storage with incremental backup
categories:
- RedShift
tags:
- aws
- redshift
- sql
- s3
image: "/assets/Export RedShift System Tables And Views To S3.jpg"
---
RedShift's system tables and views are haveing more depth information about the queries, Its highly important to export the RedShift system tables and views (STL tables) to S3 for persistent. These tables contains the information like query history, plan, query summary, etc. But these informations only available for very shot period of time. RedShift documentation says that from 2 days to 5 days we can retrive the data. It'll automatically flush the older data to keep the disk space utilization under the control. This is fine for ad-hoc performance tuning but if you want to keep the complete history of the data from these system tables and views then we have to export/Unload them to S3. I did google for implementing this on my infra, I found 2 awesome links from official AWS blog and github repo. But I wanted to try something easy with cost benifical. So I have written a stored procedure to do this.

## Exsiting Options:

* **[#1 Export via Glue](https://aws.amazon.com/blogs/big-data/how-to-retain-system-tables-data-spanning-multiple-amazon-redshift-clusters-and-run-cross-cluster-diagnostic-queries/)** - We have to spend money for Glue and DynamoDB. Lambda will be the trigger to export this.
* **[#2 Export via Lambda](https://github.com/awslabs/amazon-redshift-utils/tree/master/src/SystemTablePersistence)** - This is just a stright farword unload command, but this repo is having a lot of other utilities. If someone is new to Python, they might get stuck in understand this. But anyhow we only need the system table export option.
* **#3 Export via Stored Procedure** - I decided to use stored procedures to do this. Few tables like `stl_querytext` and `stl_query_metrics` or not having any timestamp column, but we have to export them incrementally.   

## List of tables and views:

For now I have created this procedure to export only the following tables.
1. stl_query
2. stl_dlltext
3. stl_querytext
4. stl_utilitytext
5. stl_wlm_query
6. stl_explain
7. svl_query_summary
8. stl_query_metrics
9. stl_alert_event_log

## Incremental approach: 

From the above tables, 4 tables are having the timestamp column. So we can easily find the incremental data. 
First time it'll upload all of the rows and mark the maximum value for the timestamp into this history table. 
Next time it'll go and look at the max value of the timestamp column and export the incremental data.
The tables that doesn't have the timestamp column, we can join them on the `userid`, `query id`, `pid`, `xid` columns with `stl_query` table. So we can get the incremetnal data. 

## Stored Procedure:

The following items are hardcoded, you need to replace them.
* **s3_path** - s3://prod-bucket/log-testing
* **iam_role** - arn:aws:iam::XXXXXXXXXX:role/Red-Shift-Access-S3
* **unload_options** - ADDQUOTES ALLOWOVERWRITE HEADER
* **delimiter** - |

> **NOTE**: `stl_querytxt` and `stl_ddltext` tables are having the new line charactors like `\n` and `\r`, so we have to replace them.

### Create the history table:
{% highlight sql%}
CREATE TABLE PUBLIC.history_sysobjects 
  ( 
     id               INT IDENTITY(1, 1), 
     export_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP, 
     object_name      VARCHAR(100), 
     start_time       VARCHAR(100), 
     unload_query     VARCHAR(65000) 
  ); 
{% endhighlight %}

### RedShift procedure:
{% highlight sql%}
CREATE OR REPLACE PROCEDURE public.persist_sysobjects()
LANGUAGE plpgsql AS
$$
DECLARE
-- Store the max Timestamp
stlq_starttime timestamp;
ddlt_starttime timestamp;
utlt_starttime timestamp;
wq_starttime timestamp;

-- S3 path and credentials
s3_path varchar(300);
iam_role varchar(300);
unload_options varchar(500);
delimiter varchar(10);
un_year varchar(10);
un_month varchar(10);
un_day varchar(10);

-- Declare queries as variable
stl_query_unload varchar(65000);
stl_ddltext_unload varchar(65000);
stl_utilitytext_unload varchar(65000);
stl_querytext_unload varchar(65000);
stl_wlm_query_unload varchar(65000);
stl_explain_unload varchar(65000);
svl_query_summary_unload varchar(65000);
stl_query_metrics_unload varchar(65000);
stl_alert_event_log_unload varchar(65000);

BEGIN

-- Get the yyyy/mm/dd for paritions in S3
select to_char(GETDATE(),'YYYY') into un_year;
select to_char(GETDATE(),'MM') into un_month;
select to_char(GETDATE(),'DD') into un_day;

-- S3 Options
s3_path:='s3://prod-bucket/log-testing';
iam_role:='arn:aws:iam::XXXXXXXXXX:role/Red-Shift-Access-S3';
unload_options:='ADDQUOTES ALLOWOVERWRITE HEADER';
delimiter:='|';

-- Find the current max timestamp 
select max(starttime) from stl_query into stlq_starttime;
select max(starttime) from stl_ddltext into ddlt_starttime;
select max(starttime) from stl_utilitytext into utlt_starttime;
select max(service_class_start_time) from stl_wlm_query into wq_starttime;

-- Start exporting the data
RAISE info 'Starting to unload stl_query';
stl_query_unload='unload(\'SELECT stlq.* FROM stl_query  stlq,   (SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'') h WHERE stlq.starttime > h.max_starttime\') to \''||s3_path||'/stl_query/'||un_year||'/'||un_month||'/'||un_day||'/stl_query_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_query_unload;     
insert into public.history_sysobjects (object_name,start_time,unload_query) values ('stl_query',stlq_starttime,stl_query_unload);

RAISE info 'Starting to unload stl_ddltext';
stl_ddltext_unload='unload(\'SELECT ddlt.userid,ddlt.pid,ddlt.label,ddlt.starttime, ddlt.endtime,ddlt.sequence,replace(replace(ddlt.text,\\''\\\\\\\\n\\'',\\''\\''),\\''\\\\\\\\r\\'',\\''\\'') as text FROM stl_ddltext ddlt,   (SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_ddltext\\'') h WHERE ddlt.starttime > h.max_starttime\') to \''||s3_path||'/stl_ddltext/'||un_year||'/'||un_month||'/'||un_day||'/stl_ddltext_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_ddltext_unload;     
insert into public.history_sysobjects (object_name,start_time,unload_query) values ('stl_ddltext',ddlt_starttime,stl_ddltext_unload);

RAISE info 'Starting to unload stl_utilitytext';
stl_utilitytext_unload='unload(\'SELECT utlt.* FROM stl_utilitytext utlt, (SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_utilitytext\\'') h WHERE utlt.starttime > h.max_starttime\') to \''||s3_path||'/stl_utilitytext/'||un_year||'/'||un_month||'/'||un_day||'/stl_utilitytext_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_utilitytext_unload;     
insert into public.history_sysobjects (object_name,start_time,unload_query) values ('stl_utilitytext',utlt_starttime,stl_utilitytext_unload);

RAISE info 'Starting to unload stl_querytext';
stl_querytext_unload='unload(\'select qtxt.userid,qtxt.xid,qtxt.pid,qtxt.query,qtxt.sequence,replace(replace(qtxt.text,\\''\\\\\\\\n\\'',\\''\\''),\\''\\\\\\\\r\\'',\\''\\'') as text from stl_querytext qtxt join stl_query qt on qt.userid=qtxt.userid and qt.query=qtxt.query and qt.xid=qtxt.xid and qt.pid=qtxt.pid where qt.starttime>(SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'')\') to \''||s3_path||'/stl_querytext/'||un_year||'/'||un_month||'/'||un_day||'/stl_querytext_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_querytext_unload;     
insert into public.history_sysobjects (object_name,unload_query) values ('stl_querytext',stl_querytext_unload);

RAISE info 'Starting to unload stl_wlm_query';
stl_wlm_query_unload='unload(\'SELECT wq.* FROM stl_wlm_query wq, (SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_service_class_start_time FROM public.history_sysobjects where object_name=\\''stl_wlm_query\\'') h WHERE wq.service_class_start_time > h.max_service_class_start_time\') to \''||s3_path||'/stl_wlm_query/'||un_year||'/'||un_month||'/'||un_day||'/stl_wlm_query_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_wlm_query_unload;     
insert into public.history_sysobjects (object_name,start_time,unload_query) values ('stl_wlm_query',stlq_starttime,stl_wlm_query_unload);

RAISE info 'Starting to unload stl_explain';
stl_explain_unload='unload(\'select exp.* from stl_explain exp join stl_query stlq on stlq.userid=exp.userid and stlq.query=exp.query where stlq.starttime>(SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'')\') to \''||s3_path||'/stl_explain/'||un_year||'/'||un_month||'/'||un_day||'/stl_explain_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_explain_unload;     
insert into public.history_sysobjects (object_name,unload_query) values ('stl_explain',stl_explain_unload);

RAISE info 'Starting to unload svl_query_summary';
svl_query_summary_unload='unload(\'select qs.* from svl_query_summary qs join stl_query stlq on stlq.userid=qs.userid and stlq.query=qs.query  where stlq.starttime>(SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'')\')  to \''||s3_path||'/svl_query_summary/'||un_year||'/'||un_month||'/'||un_day||'/svl_query_summary_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute svl_query_summary_unload;     
insert into public.history_sysobjects (object_name,unload_query) values ('svl_query_summary',svl_query_summary_unload);


RAISE info 'Starting to unload stl_query_metrics';
stl_query_metrics_unload='unload(\'select qm.* from stl_query_metrics qm join stl_query stlq on stlq.userid=qm.userid and stlq.query=qm.query  where stlq.starttime>(SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'')\') to \''||s3_path||'/stl_query_metrics/'||un_year||'/'||un_month||'/'||un_day||'/stl_query_metrics_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_query_metrics_unload;     
insert into public.history_sysobjects (object_name,unload_query) values ('stl_query_metrics',stl_query_metrics_unload);

RAISE info 'Starting to unload stl_alert_event_log';
stl_alert_event_log_unload='unload(\'select evnt.* from stl_alert_event_log evnt join stl_query qt on qt.userid=evnt.userid and qt.query=evnt.query and qt.xid=evnt.xid and qt.pid=evnt.pid  where qt.starttime>(SELECT NVL(MAX(start_time),\\''1902-01-01\\''::TIMESTAMP) AS max_starttime FROM public.history_sysobjects where object_name=\\''stl_query\\'')\') to \''||s3_path||'/stl_alert_event_log/'||un_year||'/'||un_month||'/'||un_day||'/stl_alert_event_log_\' iam_role \''||iam_role||'\' '||unload_options||' delimiter \''||delimiter||'\'';

execute stl_alert_event_log_unload;     
insert into public.history_sysobjects (object_name,unload_query) values ('stl_alert_event_log',stl_alert_event_log_unload);

END
$$; 
{% endhighlight %} 

