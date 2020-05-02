---
title: Debezium MySQL Snapshot For CloudSQL(MySQL) From Replica
date: 2020-01-21 19:46:00 +0530
description: 'Debezium take snapshot from GCP CloudSQL(MySQL) from Read Replica. We
  have to create a VM replica or CloudSQL Replica for snapshot. '
categories:
- Kafka
tags:
- kafka
- debezium
- mysql
- replication
image: "/assets/Debezium MySQL Snapshot For CloudSQL(MySQL) From Replica.jpg"

---
The snapshot in Debezium will do a historical data load from the source database to the Kafka topics. But generally its not a good practice to this if you have a huge data in your tables. Recently I have published many blog posts to perform this snapshot from Read Replica(with/without GTID, AWS Aurora). One guy commented that, in GCP the MySQL managed service is called CloudSQL. There we don't have much control to stop replication, perform the modifications that we want. So how can we avoid snapshots in CloudSQL and take debezium snapshots from CloudSQL Read Replica? I have spent some time today and figured out a way to do this.

## The Approach:

We can't enable binlogs on read replica. So we have to setup an external read replica for this. If the external replica is a VM, then we can enable the `log-slave-updates` with GTID. Then we can [**follow this blog post**](https://thedataguy.in/debezium-mysql-snapshot-from-read-replica-with-gtid/) to solve this problem. But I want to solve this by using CloudSQL read replica.

* For this create a new read replica for your cloudsql.
* If you already have a read replica, then create one more (because we'll break the replication, so don't use the active one).
* Disable the replication on the new Replica and make a note of the master's binlog information.
* Then Promote the replica. So it'll automatically enable the binlog.
* Now create a connector to read data from the replica node.
* Once the snapshot is done, manually update the `connect-offset` with Master's binlog info.
* Update the connector with Master's IP address.
* Then it'll read from Master.

## Proof Of Concept:

Read my previous blog posts to install and configure Confluent Kafka connect and etc.

* CloudSQL Master IP  - 172.24.0.13
* CloudSQL Replica IP - 172.24.0.19

## Sample data:

Create a new database to test this sync and insert some values.
{% highlight sql %}
create database bhuvi;
use bhuvi;
create table rohi (
id int,
fn varchar(10),
ln varchar(10),
phone int);

insert into rohi values (1, 'rohit', 'last',87611);
insert into rohi values (2, 'rohit', 'last',87611);
insert into rohi values (3, 'rohit', 'last',87611);
insert into rohi values (4, 'rohit', 'last',87611);
insert into rohi values (5, 'rohit', 'last',87611);
{% endhighlight %}

Once your replica is in sync with Master, disable the replication from the CloudSQL Console.

![](/assets/Debezium MySQL Snapshot For CloudSQL(MySQL) From Replica1.jpg)

Then login to the Replica and get the master's binlog file name and position from the `show slave status`

{% highlight sql %}
Master_Host: 172.24.0.13
Master_User: cloudsqlreplica
Master_Port: 3306
Connect_Retry: 60
Master_Log_File: mysql-bin.000001
Read_Master_Log_Pos: 10259755
Relay_Log_File: relay-log.000002
Relay_Log_Pos: 26567
Relay_Master_Log_File: mysql-bin.000001
Slave_IO_Running: No
Slave_SQL_Running: No
Last_Error:
Skip_Counter: 0
Exec_Master_Log_Pos: 10259755  
Master_UUID: c3c1d467-3bee-11ea-907b-4201ac18000c
{% endhighlight %}

* Binlog File Name - Master_Log_File
* Binlog Position - 10259755
* Maser's UUID - Master_UUID

Now promote the replica, it'll enable the binlog on this replica server.

![](/assets/Debezium MySQL Snapshot For CloudSQL(MySQL) From Replica2.jpg)

To simulate the complexity, add one more row on the master node(this row will not be replicated, since the replication is disables). So once the snapshot done, we'll switch the MySQL IP. Then it should read this new row.

{% highlight sql %}
insert into rohi values (6, 'rohit', 'last',87611);
{% endhighlight %}

## Create the MySQL Connector:

Use the below MySQL connector JSON config.  (replace the MySQL details, kafka details, and Transformation things)

{% highlight JSON %}
{
"name": "mysql-connector-db01",
"config": {
"name": "mysql-connector-db01",
"connector.class": "io.debezium.connector.mysql.MySqlConnector",
"database.server.id": "1",
"tasks.max": "3",
"database.history.kafka.bootstrap.servers": "10.128.0.12:9092",
"database.history.kafka.topic": "replica-schema-changes.mysql",
"database.server.name": "mysql-db01",
"database.hostname": "172.24.0.19",
"database.port": "3306",
"database.user": "root",
"database.password": "****",
"database.whitelist": "bhuvi",
"internal.key.converter.schemas.enable": "false",
"key.converter.schemas.enable": "false",
"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter",
"internal.value.converter.schemas.enable": "false",
"value.converter.schemas.enable": "false",
"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter",
"value.converter": "org.apache.kafka.connect.json.JsonConverter",
"key.converter": "org.apache.kafka.connect.json.JsonConverter",
"transforms": "unwrap",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.add.source.fields": "ts_ms",
"tombstones.on.delete": false
}
}
{% endhighlight %}

### Install the Connector:

    curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" http://localhost:8083/connectors -d @mysql.json

### Ingest the master binlog info:

Get the last read binlog info from the Kafka topic and manually add the master's binlog into it.

    kafkacat -b localhost:9092 -C -t connect-offsets  -f 'Partition(%p) %k %s\n'
    
    Partition(1) ["mysql-connector-db01",{"server":"mysql-db01"}] {"file":"mysql-bin.000004","pos":89114,"gtids":"4ac96fbf-3c4b-11ea-8ea0-4201ac180012:1-19,c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}

Now replica the binlog file name, position,  UUID. (sometimes you'll get multiple UUID, so remove everything, just keep the master's UUID)

    echo '["mysql-connector-db01",{"server":"mysql-db01"}]|{"file":"mysql-bin.000001","pos":10259755,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}' | kafkacat -P -b localhost -t connect-offsets -K \| -p 1

Check this data in `connect-offsets`
{% highlight shell %}
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-offsets --from-beginning
{"file":"mysql-bin.000004","pos":67739,"gtids":"4ac96fbf-3c4b-11ea-8ea0-4201ac180012:1-237,c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}

{"file":"mysql-bin.000001","pos":10259755,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}
{% endhighlight %}

### Update the connector:

Use the below Config file to update it. Make sure the `database.history.kafka.topic` will be a new topic and`snapshot.mode` must be `SCHEMA_ONLY_RECOVERY`. And the update the MySQL details, kafka.
{% highlight JSON %}
{
"connector.class": "io.debezium.connector.mysql.MySqlConnector",
"snapshot.locking.mode": "none",
"tasks.max": "3",
"database.history.kafka.topic": "master-schema-changes.mysql",
"transforms": "unwrap",
"internal.key.converter.schemas.enable": "false",
"transforms.unwrap.add.source.fields": "ts_ms",
"tombstones.on.delete": "false",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"value.converter": "org.apache.kafka.connect.json.JsonConverter",
"database.whitelist": "bhuvi",
"key.converter": "org.apache.kafka.connect.json.JsonConverter",
"database.user": "root",
"database.server.id": "1",
"database.history.kafka.bootstrap.servers": "10.128.0.12:9092",
"database.server.name": "mysql-db01",
"database.port": "3306",
"key.converter.schemas.enable": "false",
"internal.key.converter": "org.apache.kafka.connect.json.JsonConverter",
"database.hostname": "172.24.0.13",
"database.password": "****",
"internal.value.converter.schemas.enable": "false",
"name": "mysql-connector-db01",
"value.converter.schemas.enable": "false",
"internal.value.converter": "org.apache.kafka.connect.json.JsonConverter",
"snapshot.mode": "SCHEMA_ONLY_RECOVERY"
}
{% endhighlight %}

Once its updated, check the status of the connector.

{% highlight shell %}
curl GET localhost:8083/connectors/mysql-connector-db01/status

{
"name": "mysql-connector-db01",
"connector": {
"state": "RUNNING",
"worker_id": "10.128.0.14:8083"
},
"tasks": \[
{
"id": 0,
"state": "RUNNING",
"worker_id": "10.128.0.14:8083"
}
\],
"type": "source"
}
{% endhighlight %}

Check the binlog inforation from the connect-offset topic.
{% highlight shell %}
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-offsets --from-beginning

{"file":"mysql-bin.000004","pos":67739,"gtids":"4ac96fbf-3c4b-11ea-8ea0-4201ac180012:1-237,c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}
{"file":"mysql-bin.000001","pos":10259755,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}

{"ts_sec":1579610781,"file":"mysql-bin.000001","pos":10263525,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36016","row":1,"server_id":128423640,"event":2}
{% endhighlight %}

You can see the thrid line which is nothing but the 6th row that we inserted on master after disabling the replication. Now insert one more row and see whether is read by the connector or not.
{% highlight sql %}
mysql> insert into rohi values (7, 'rohit', 'last',87611);
{% endhighlight %}

{% highlight shell %}
kafka-console-consumer --bootstrap-server localhost:9092 --topic connect-offsets --from-beginning
{"file":"mysql-bin.000004","pos":67739,"gtids":"4ac96fbf-3c4b-11ea-8ea0-4201ac180012:1-237,c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}
{"file":"mysql-bin.000001","pos":10259755,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36003"}

{"ts_sec":1579610781,"file":"mysql-bin.000001","pos":10263525,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36016","row":1,"server_id":128423640,"event":2}

{"ts_sec":1579612311,"file":"mysql-bin.000001","pos":10350729,"gtids":"c3c1d467-3bee-11ea-907b-4201ac18000c:1-36322","row":1,"server_id":128423640,"event":2}
{% endhighlight %}

### Check the data in the table's data from Kafaka

{% highlight shell %}
kafka-console-consumer --bootstrap-server localhost:9092 --topic mysql-db01.bhuvi.rohi --from-beginning
{"id":1,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":0}
{"id":2,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":0}
{"id":3,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":0}
{"id":4,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":0}
{"id":5,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":0}
{"id":6,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":1579610781000}
{"id":7,"fn":"rohit","ln":"last","phone":87611,"__ts_ms":1579612311000}
{% endhighlight %}

### Debezium Series blogs:

1. [Build Production Grade Debezium Cluster With Confluent Kafka](https://thedataguy.in/build-production-grade-debezium-with-confluent-kafka-cluster/)
2. [Monitor Debezium MySQL Connector With Prometheus And Grafana](https://thedataguy.in/monitor-debezium-mysql-connector-with-prometheus-and-grafana/)
3. [Debezium MySQL Snapshot From Read Replica With GTID](https://thedataguy.in/debezium-mysql-snapshot-from-read-replica-with-gtid/)
4. [Debezium MySQL Snapshot From Read Replica And Resume From Master](https://thedataguy.in/debezium-mysql-snapshot-from-read-replica-and-resume-from-master/)
5. [Debezium MySQL Snapshot For AWS RDS Aurora From Backup Snaphot](https://thedataguy.in/debezium-mysql-snapshot-for-aws-rds-aurora-from-backup-snaphot/)
6. [RealTime CDC From MySQL Using AWS MSK With Debezium](https://medium.com/searce/realtime-cdc-from-mysql-using-aws-msk-with-debezium-28da5a4ca873)