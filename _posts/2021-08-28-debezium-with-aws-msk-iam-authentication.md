---
layout: post
date: 2021-08-28 11:10:00 +0530
title: Debezium With AWS MSK IAM Authentication
description: Integrate Debezium With AWS MSK IAM Authentication. We can control and track all the Kafka level access with IAM and cloudtrail.
categories:
- AWS
tags:
- aws
- kafka
- debezium
- iam
- security
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/aws/Debezium With AWS MSK IAM Authentication.jpg"
---
AWS MSK now supports Kafka ACL via IAM permissions. You can assign that permission to an IAM user(`aws credentials file`) or an IAM role that is attached to the Kafka Clients(Both producers and consumers). These IAM permissions are having similar names as the Kafka ACLs. All the activites like Create topic, delete topic, describe topic can be logged into the cloudtrail logs. We can't expect all the ACLs, but almost all the necessary permissions on the Topic level are already available. Soon we can expect that they can add more permissions. In this blog, we are going to see how to integrate the Debezium with AWS MSK IAM authentication and some problems I faced while implementing this. 

## How it works?

{% include lazyload.html image_src="/assets/aws/Debezium With AWS MSK IAM Authentication.jpg" image_alt="Debezium With AWS MSK IAM Authenticatio" image_title="Debezium With AWS MSK IAM Authenticatio" %}

* The integration between Kafka ACL and the IAM will be established by the open-source [msk-iam-library](https://github.com/aws/aws-msk-iam-auth) which is developed by the AWS. 
* This plugin will convert the IAM policies to Kafka ACLs internally(not exactly the same, but the underlying mechanism is to map with Kafka ACL).
* Once the request is generated then it'll make an API call to AWS IAM and then check the permissions, if all are matching, then the request will be allowed to perform the actions on the Kafka cluster. Else it'll return the error.
* We can use both aws credentials profile or directly attach an IAM role to that Kafka client. 

## Let's configure

* I want to manage the IAM permission in a more simple way, so Im using the prefix `debezium-` for everything(all of my debezium topics and the kafa connect storage - offset, config, status). So I need to grant the Read and Write permission with this prefix. 
* My sink connectors are named with `debezium-` prefix. I have mentioned an issue I faced with sink connectors in the caveat section. 
* Kafka connect worker group id is `debezium-cluster`.

Download the AWS MSK IAM library into the Kafka connect instance. (always check their [repo]() for the latest version of the library.
```bash
mkdir /opt/kafka-custom-lib/
cd /opt/kafka-custom-lib/
wget https://github.com/aws/aws-msk-iam-auth/releases/download/1.1.0/aws-msk-iam-auth-1.1.0-all.jar
```
Edit the Kafka run class file(`/usr/bin/kafka-run-class`) and the library location under the `launch mode` section then restart the Kafka connect service.
```bash
-- From this
# Launch mode
if [ "x$DAEMON_MODE" = "xtrue" ]; then
  nohup "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS  $KAFKA_OPTS "$@" > "$CONSOLE_OUTPUT_FILE" 2>&1 < /dev/null &
else
  exec "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS  $KAFKA_LOG4J_OPTS  $KAFKA_OPTS "$@"
fi

-- To this
# Launch mode
if [ "x$DAEMON_MODE" = "xtrue" ]; then
  nohup "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH:"/opt/kafka-custom-lib/*"  $KAFKA_OPTS "$@" > "$CONSOLE_OUTPUT_FILE" 2>&1 < /dev/null &
else
  exec "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS -cp $CLASSPATH:"/opt/kafka-custom-lib/*" $KAFKA_LOG4J_OPTS  $KAFKA_OPTS "$@"
fi
```
Create an EC2 IAM role and attach it to the Kafka connect instance(you can read [the AWS doc](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#working-with-iam-roles) to create the role)

Create attach the inline policy to that EC2 IAM Role. Make sure you replace the following with your cluster details. Read more from [this doc](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html#kafka-actions) to know better about this syntax

* REGION
* ACCOUNT_ID
* CLUSTER_NAME
* CLUSTER_UUID
* debezium-*(worker group id)
* debezium* (Kafka topic prefix)
* connect-debezium* (Sink connector prefix)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:WriteDataIdempotently",
                "kafka-cluster:ReadData",
                "kafka-cluster:DescribeTransactionalId",
                "kafka-cluster:AlterTransactionalId",
                "kafka-cluster:DescribeTopicDynamicConfiguration",
                "kafka-cluster:AlterTopicDynamicConfiguration",
                "kafka-cluster:AlterGroup",
                "kafka-cluster:AlterTopic",
                "kafka-cluster:CreateTopic",
                "kafka-cluster:DescribeTopic",
                "kafka-cluster:DescribeGroup",
                "kafka-cluster:DescribeClusterDynamicConfiguration",
                "kafka-cluster:Connect",
                "kafka-cluster:WriteData"
            ],
            "Resource": [
                "arn:aws:kafka:REGION:ACCOUNT_ID:topic/CLUSTER_NAME/CLUSTER_UUID/debezium*",
                "arn:aws:kafka:REGION:ACCOUNT_ID:cluster/CLUSTER_NAME/CLUSTER_UUID",
                "arn:aws:kafka:REGION:ACCOUNT_ID:transactional-id/CLUSTER_NAME/CLUSTER_UUID/*",
                "arn:aws:kafka:REGION:ACCOUNT_ID:group/CLUSTER_NAME/CLUSTER_UUID/debezium-*",
                "arn:aws:kafka:REGION:ACCOUNT_ID:group/CLUSTER_NAME/CLUSTER_UUID/connect-debezium*"
            ]
        }
    ]
}
```
Copy the default Java `cacerts` file as a Truststore file. Because this method requires a TLS connection from the client. So we can use the default Java certificate. Im running my Kafka connect with Amazon corretto. 
```bash
cp /usr/lib/jvm/java-1.8.0-amazon-corretto.x86_64/jre/lib/security/cacerts /opt/ssl/kafka.client.truststore.jks
```
(**Optional**) To make the file more secure, set read-only permission to that file. Im running the kafka connect service under the `cp-kafka` user `confluent` as the group.
```bash
chown cp-kafka:confluent /opt/ssl/kafka.client.truststore.jks
chmod 400 /opt/ssl/kafka.client.truststore.jks
```
Add the following properties into the worker properties file and restart the service. Im using distributed properties file.
```bash
vim /etc/kafka/connect-distributed.properties

## SSL Properties
ssl.truststore.location=/opt/ssl/kafka.client.truststore.jks
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler	

## SSL Properties for Producer app
producer.ssl.truststore.location=/tmp/kafka.client.truststore.jks
producer.security.protocol=SASL_SSL
producer.sasl.mechanism=AWS_MSK_IAM
producer.sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
producer.sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler

## SSL Properties for Consumer app
consumer.ssl.truststore.location=/tmp/kafka.client.truststore.jks
consumer.security.protocol=SASL_SSL
consumer.sasl.mechanism=AWS_MSK_IAM
consumer.sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
consumer.sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
``` 
> It's mandatory to add the producer and consumer properties into this file, else it'll throw an error.

(Optional but applicable for t3 MSK family) - If you are running your MSK cluster, then while connecting with IAM will create more requests and your cluster may block those request and your client will return `Too many connects` error. So lets add this line into your worker properties file. If need use a larger value than this. Read the thread [here](https://github.com/aws/aws-msk-iam-auth/issues/28)
```bash
reconnect.backoff.ms=100000
```
Its time for the debezium connector configuration. Lets add these lines into your debezium JSON config file.
```json
"database.history.security.protocol":"SASL_SSL",
"database.history.sasl.mechanism":"AWS_MSK_IAM",
"database.history.sasl.jaas.config":"software.amazon.msk.auth.iam.IAMLoginModule required;",
"database.history.sasl.client.callback.handler.class":"software.amazon.msk.auth.iam.IAMClientCallbackHandler",

"database.history.producer.security.protocol":"SASL_SSL",
"database.history.producer.sasl.mechanism":"AWS_MSK_IAM",
"database.history.producer.sasl.jaas.config":"software.amazon.msk.auth.iam.IAMLoginModule required;",
"database.history.producer.sasl.client.callback.handler.class":"software.amazon.msk.auth.iam.IAMClientCallbackHandler",

"database.history.consumer.security.protocol":"SASL_SSL",
"database.history.consumer.sasl.mechanism":"AWS_MSK_IAM",
"database.history.consumer.sasl.jaas.config":"software.amazon.msk.auth.iam.IAMLoginModule required;",
"database.history.consumer.sasl.client.callback.handler.class":"software.amazon.msk.auth.iam.IAMClientCallbackHandler",
```
> Again producer and consumer sections are mandatory here as well, else you'll get an error.

Deploy the connector.
```bash
curl -X POST -H "Accept: application/json" -H "Content-Type: application/json" http://localhost:8083/connectors -d @mysql_iam.json
```
Check the status.
```bash
curl GET localhost:8083/connectors/mysql-connector-02/status | jq
{
  "name": "mysql-connector-02",
  "connector": {
    "state": "RUNNING",
    "worker_id": "172.30.32.13:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "172.30.32.13:8083"
    }
  ],
  "type": "source"
}
```
Check the topics created by debezium.
```bash
kafka-topics --list --zookeeper ZOOKEEPER_ENDPOINT:2181

debezium-connect-configs
debezium-connect-offsets
debezium-connect-status
debezium-mysql-db01
debezium-mysql-db01.bb.iam6
debezium-mysql-db01.bb.schema-changes.mysql
debezium-mysql-db01.bb.tt
debezium-mysql-db01.bb.tt1
```
Yeah, it worked, lets see the activities on the CloudTrail console

{% include lazyload.html image_src="/assets/aws/Debezium With AWS MSK IAM Authentication-console.jpg" image_alt="Debezium With AWS MSK IAM Authenticatio" image_title="Debezium With AWS MSK IAM Authenticatio" %} 

## Bonus Tip: Access AWS MSK Zookeeper with TLS

The TLS auth will be implemented between brokers, client to brokers, zookeeper to brokers. But client to Zookeeper can be accessed without TLS. Im sharing some steps to securly access the zookeeper with TLS.

We already have the truststore file. And it has password protected. The default password is `changeit`. Debezium doesn't need this password, but to connect with zookeeper its required. So let reset this password with a complex one.

```bash
keytool -storepasswd -keystore /opt/ssl/kafka.client.truststore.jks  -storepass changeit -new STRONG_PASS
```
Then store this password into the AWS secret manager. I have stored it in the name of `dev/debezium/jkspassword` and the key value pair is 
```json
{
  "jkspass": "STRONG_PASS"
}
```
Im using `lenses`'s secret provider plugin to retrive the secrects in Kafka connect. Read the [steps here](https://thedataguy.in/integrate-debezium-with-aws-secret-manager-for-retrieving-passwords/) to install and configure it.

This secrect provide will map a file in your Kafka config directory. We need to create that file manually. The file path is same as your AWS secret name. Or create this as folder. 

```bash
cd /etc/kafka
touch dev/debezium/jkspassword
```

Add the zookeeper SSL properties into the workder.properties file(im using distributed properties), then restart the worker service.
```bash
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty 
zookeeper.client.secure=true 
zookeeper.ssl.trustStore.location=/opt/ssl/kafka.client.truststore.jks
zookeeper.ssl.trustStore.password=${aws:dev/debezium/jkspassword:jkspass}
```

Now lets see the access with TLS Port(2182)
```bash
kafka-topics --list --zookeeper z-3.kafka.ap-south-1.amazonaws.com:2182,z-1.kafka.ap-south-1.amazonaws.com:2181,z-2.kafka.ap-south-1.amazonaws.com:2182

__amazon_msk_canary
__amazon_msk_canary_state
__consumer_offsets
debezium-connect-configs
debezium-connect-offsets
debezium-connect-status
debezium-mysql-db01
debezium-mysql-db01.bb.iam6
debezium-mysql-db01.bb.schema-changes.mysql
debezium-mysql-db01.bb.tt
debezium-mysql-db01.bb.tt1
```
Lets talk about some issues I faced.

## Caveats:

### #1 Cluster type:

Initially, I was using t3.small. So I used a very larger value for `reconnect.backoff.ms` But it didn't work for me, Then I increased the cluster to m5. After that, it was fixed. Then again I scale down to t3. This time I didn't get that `Too many connects` error. I don't why after scale down it didn't occur. 

### #2 No errors on the connector even its not producing any data

In the worker properties file, I didn't add the `producer.ssl*` and `consumer.ssl*`. It'll not work, so it has to return errors while deploying debzium or sink connectors. But status is showing running and the kafka connect logs returned some errors like disconnected.

```bash
Aug 26 14:54:16 ip-172-30-32-13 connect-distributed: [2021-08-26 14:54:16,708] WARN [debezium-s3-sink-db01|task-0] [Consumer clientId=connector-consumer-debezium-s3-sink-db01-0, groupId=connect-debezium-s3-sink-db01] Bootstrap broker b-1.kafka.ap-south-1.amazonaws.com:9098 (id: -2 rack: null) disconnected

Aug 26 14:59:15 ip-172-30-32-13 connect-distributed: [2021-08-26 14:59:15,548] WARN [mysql-connector-02|task-0] [Producer clientId=connector-producer-mysql-connector-02-0] Bootstrap broker b-1.kafka.ap-south-1.amazonaws.com:9098 (id: -2 rack: null) disconnected
```

### #3 Addon to #2 issue

We added the `producer.ssl*` and `consumer.ssl*` to the worker properties file, but in the debezium connector JSON file, I tried without adding the `database.history.producer.*` and `database.history.consumer.*` properties. After the deployment, the connector was running but nothing gets produced. But the log was showing,

```bash
Aug 26 14:59:15 ip-172-30-32-13 connect-distributed: [2021-08-26 14:59:15,548] WARN [mysql-connector-02|task-0] [Producer clientId=connector-producer-mysql-connector-02-0] Bootstrap broker b-1.kafka.ap-south-1.amazonaws.com:9098 (id: -2 rack: null) disconnected
```
### #4 Sink connector group

I granted the IAM topic level permissions to debezium-* prefix. But when I deployed the sink connector with the name `s3-sink-conn-01`, It threw an error like Access Denied on the group `connect-s3-sink-conn-01`. 

```bash
GroupAuthorizationException: Not authorized to access group: connect-s3-sink-conn-01
```
Because all the sink connectors will use a dedicated consumer group that is named `connect-SINK_CONNECTOR_NAME`. But I granted the IAM permission with debezium only. So I had to use a different prefix for the Sink connectors in IAM permission. Then I started using the Sink connectors names with debezium-* prefix. And in the IAM I have granted the permission as like below.

```json
"arn:aws:kafka:REGION:ACCOUNT_ID:group/CLUSTER_NAME/CLUSTER_UUID/connect-debezium*"
```

### #5 zookeeper with ACL

I tried to delete a topic using 2 methods. 1st one with --bootstap flag and the other one with --zookeeper.

```bash
kafka-topics \ 
--bootstrap-server b-1.kafka.ap-south-1.amazonaws.com:9098,b-2.kafka.ap-south-1.amazonaws.com:9098 \ 
--delete \ 
--topic debezium-my-topic \ 
--command-config /etc/kafka/client.properties 

kafka-topics \ 
--zookeeper z-3.kafka.ap-south-1.amazonaws.com:2182,z-1.kafka.ap-south-1.amazonaws.com:2181,z-2.kafka.ap-south-1.amazonaws.com:2181 \ 
--delete \ 
--topic debezium-my-topic
```
According to my IAM policy(mentioned above), `DeleteTopic` is not granted, so the 1st command returned the IAM permission denied error. But when I call the zookeeper to delete the topic, it actually deleted.  Because IAM auth will not enforce the zookeeper nodes. It can bypass. Anyhow zookeeper will be removed in future Kafka releases. 

### #6 Zookeeper security

The only way to solve the above issue is, create a seperate Security group for zookeeper and allow only Kafka broker nodes IP address into that. And if you want to access zookeeper then you can add your IP into that security group in on-demand basis. So the kafka clients will not have access to the zookeeper. These steps are documented [here](https://docs.aws.amazon.com/msk/latest/developerguide/zookeeper-security.html)

### #7 Few more things that need attention:

* You cannot enable the IAM auth on the running cluster.
* Once you enabled the IAM auth, then you cannot change the Auth method to SASA or Mutual TLS or Disable. 
* IAM auth will not work in Zookeeper nodes, its only applicable for the brokers.
* Once you enable the IAM, then the clients must use TLS auth, without TLS you'll get a connection timeout error. 

## Reference Links:

1. [A blog from AWS - how to use IAM with kafka client](https://aws.amazon.com/blogs/big-data/securing-apache-kafka-is-easy-and-familiar-with-iam-access-control-for-amazon-msk/)
2. [Implementing Kafka IAM permissions - AWS docs](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html)
3. [IAM Permissions will not work with Zookeeper - my SO question](https://stackoverflow.com/questions/68941629/kafka-delete-a-topic-using-bootstrap-server-vs-zookeeper)
4. [Scalability of Kafka Messaging using Consumer Groups](https://blog.cloudera.com/scalability-of-kafka-messaging-using-consumer-groups/)