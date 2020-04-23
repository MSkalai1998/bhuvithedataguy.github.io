---
title: 3 Things Before Migrating To CloudSQL(MySQL)
date: 2020-03-20 00:10:00 +0530
description: 'Make a note of these 3 things before your migrating your MySQL database to GCP CloudSQL'
categories:
- GCP
tags:
- gcp
- mysql
- cloudsql
- migration
image: "/assets/Debezium MySQL Snapshot For CloudSQL(MySQL) From Replica.jpg"

---
If you are going to migrate your MySQL workloads to GCP's managed database service CloudSQL, then you have to keep these points in mind. We have done a lot of CloudSQL migrations. But sometimes it's not smooth as we thought. Generally, people don't even think that these thinks will make the replication failure. I listing 3 things that ate our brain and time while migrating to CloudSQL.

## 1. Server character set:
CloudSQL by default using `utf8` as the server character set. But it is customizable, we can change it any time. But still, it'll mess up your application later. We had a MySQL server on a VM where the server's character set was `latin1`. We dump the database and restore it to CloudSQL. While launching the CloudSQL we didn't set up any Database flags. So the data restore with `utf8` character set. 

**Before Migration**

{% highlight sql %}
mysql> SHOW SESSION VARIABLES LIKE 'character\_set\_%';

+--------------------------+--------+
| Variable_name            | Value  |
+--------------------------+--------+
| character_set_client     | utf8   |
| character_set_connection | utf8   |
| character_set_database   | latin1 |
| character_set_filesystem | binary |
| character_set_results    | utf8   |
| character_set_server     | latin1 |
| character_set_system     | utf8   |
+--------------------------+--------+
{% endhighlight %}

**After Migration**

{% highlight sql %}
mysql>  SHOW SESSION VARIABLES LIKE 'character\_set\_%';
+--------------------------+--------+
| Variable_name            | Value  |
+--------------------------+--------+
| character_set_client     | utf8   |
| character_set_connection | utf8   |
| character_set_database   | utf8   |
| character_set_filesystem | binary |
| character_set_results    | utf8   |
| character_set_server     | utf8   |
| character_set_system     | utf8   |
+--------------------------+--------+
{% endhighlight %}

We had a table which has the data in Japanise language.  But after cutover, we were not able to get the exact data. 
**Actual Data**
```
サッポロプレミアム生ビール（中ジョッキ) 
```
**After the migration**
```
ã‚µãƒƒãƒ�ãƒ­ãƒ—ãƒ¬ãƒŸã‚¢ãƒ ç”Ÿãƒ“ãƒ¼ãƒ«ï¼ˆä¸­ã‚¸ãƒ§ãƒƒã‚­)
```

### Things we tried to solve but didn't work:
1. Changed the `character_set_server` to `latin1`.
2. Dump the table from the same server with `--default-character-set=latin1` and restored.
3. Dump the table from the old MySQL server with `--default-character-set=latin1` and restored the dump again with `--default-character-set=latin1`.
4. Created a new replica with `character_set_server = latin1`

### What we did as a Workaround:
From the application, change the default client connection to `latin1`.  But please make sure the character server character set before the migration.

## 2. MyISAM tables:
Everybody aware that CloudSQL only supports the `innodb` engine. But the CloudSQL migration option will help you to convert the MyISAM engine to Innodb during the restore. We had a very large table. So we let the conversion with CloudSQL. Once the data had been restored then the replication started without any issues. But After a few hours, we noticed that the replication is broken due to Duplicate Key conflict. We got an error message like below.

{% highlight sql %}
Could not execute Write_rows event on table dbname.tablename; Duplicate entry '32640' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000576, end_log_pos 16529819
{% endhighlight %}
We saw that the table had `auto_increment` primary key, also it was working fine for a few hours. 

### Things we tried to solve but didn't work:
Nothing much tried with the latest dumps. We did this dump and restore many times, but every time it got failed. 

### Workaround:
Convert the MyISAM tables to Innodb on the master node, then take the dump file for the migration. 

## 3. Don't include the Views in your dump file:
The default dump file restore user in CloudSQL has very limited privilege, it doesn't have the `create view` permission. During the restore when it detects the create view command, then it'll stop the process. And the other bad news is it'll never log any single entry about this view in the stackdriver log. We had this situation, and we were not able to identify what was causing this issue, then we requested the GCP support team to share the internal logs which only available for them. There we got this error line information.

{% highlight sql %}
ERROR 1045 (28000) at line 71960: Access denied for user 'cloudsqlimport'@'localhost' (using password: NO)
{% endhighlight %}

When we extracted line 71960 from the dump file, we got to know that it is a view. Anyhow the GCP's documentation says don't include the view during the dump. So if you encountered to create a view or some with `definer=` those line numbers will not bere in your stackdriver logs. 

If you had some strange experience with CloudSQL migration, please leave them in the comments. 