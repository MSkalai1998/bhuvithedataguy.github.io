---
layout: post
date: 2021-07-20 17:45:00 +0530
title: Integrate Debezium with AWS Secret Manager For Retrieving Passwords
description: Integrate Debezium with AWS Secret Manager for retrieving passwords. We can use this IAM Role or Access and Secret keys. 
categories:
- Airflow
tags:
- gcp
- bigquery
- postgresql
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/Integrate Debezium with AWS Secret Manager For Retrieving Passwords.jpeg"
---

Debezium is my all-time favorate CDC tool. It's an OpenSource Kafka connect application that extracts the CDC data from MSSQL, MySQL, Oracle, Postgresql MongoDB, and a few more databases. Generally, the Kafka connect will use plain text passwords or [external secrets](https://cwiki.apache.org/confluence/display/KAFKA/KIP-297%3A+Externalizing+Secrets+for+Connect+Configurations) to pass the password. But still, it is a plain text password from the external secret. After a long time, I was going to deploy a debezium connector for a MySQL database. The whole stack is on AWS and we are using AWS Secret manager for storing and rotating passwords. I was checking a few more options to integrate the AWS Secret Manager with debezium via Environment variables, but it couldn't make it. It is very easy if you are deploying it via Docker. But I was doing this on an EC2 instance. Then I found an interesting plugin called [Secret Provider](https://github.com/lensesio/secret-provider) from [lenses.io](https://docs.lenses.io/4.1/integrations/connectors/secret-providers/). Seems it's providing the out of the box integration with AWS, Azure, Hashicorp Vault, and more. Here is a small tutorial for integrating the AWS Secret manager with Debezium using this Lenses's secret provider.

## Reasons to consider this plugin:

1. Integration with cloud providers(GCP is still not supported)
2. Passwords are not exposed via API 
3. Lightweight
4. Works with EC2 instance's IAM Role and Keys.

## How does it work?

{% include lazyload.html image_src="/assets/Integrate Debezium with AWS Secret Manager For Retrieving Passwords.jpeg" image_alt="Integrate Debezium with AWS Secret Manager For Retrieving Passwords" image_title="Integrate Debezium with AWS Secret Manager For Retrieving Passwords" %}

In any source or sink connectors, we need to pass the Secret name and it'll look like an env variable. For example, if we have a secret name called `prod_db_password` for a MySQL password with the Key value pair of `{"password": "My-Strong-Pass"}` then we need to pass the value as `${aws:prod_db_password:password}`.

Then while creating the connector, it'll check whether the IAM role has been attached and having the necessary permissions to get the secret, Or any access and secret key provide to retrieve the secret then, it'll call the `GetSecret` API to get the password from the AWS Secret Manager. 

## Demo Time:

Let's create a secret for your MySQL user in the AWS secret.

* Secret Name: `prod/debezium/mysql/service1`
* Key & Value: `mysql_pass:mystrong-pass`

## Create an EC2 IAM role:

### Permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:<<YOUR_SECRET_REGION>>:<<AWS_ACCOUNT_NUMBER>>:secret:prod/debezium/*"
        }
    ]
}
```

## Install the Secret Provider plugin:

```bash
cd /usr/share/confluent-hub-components
wget https://github.com/lensesio/secret-provider/releases/download/2.1.6/secret-provider-2.1.6-all.jar
```

> Note:  `/usr/share/confluent-hub-components` - This is my plugin directory, you can download the JAR anywhere and put that path into `plugin.path`

Add the plugin configurations into the worker properties

```bash
vi /etc/kafka/connect-distributed.properties

config.providers=aws
config.providers.aws.class=io.lenses.connect.secrets.providers.AWSSecretProvider
config.providers.aws.param.aws.auth.method=default
config.providers.aws.param.aws.access.key=$AWS_ACCESS_KEY_ID
config.providers.aws.param.aws.secret.key=$AWS_SECRET_ACCESS_KEY
config.providers.aws.param.aws.region=ap-south-1
config.providers.aws.param.file.dir= /etc/kafka/
```
Restart the kafka connect service.

## Debezium Connector Config: 

Its not a full config file. File name: `mysql.json`

```json
...
...
"database.hostname": "xxxxxx.ap-south-1.rds.amazonaws.com",
"database.port": "3306",
"database.user": "root",
"database.password": "${aws:prod/debezium/mysql/service1:mysql_pass}",
```

## Testing:

Let's create this connector.
```bash
curl -X POST -H "Content-Type: application/json" http://localhost:8083/connectors -d @mysql.json
```
And the status:
```json
curl GET localhost:8083/connectors/mysql-connector-db01/status 

{
  "name": "mysql-connector-db01",
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

Whoo hooo!, it worked. I really like this plugin but still, one thing is not clear to me. While calling the secret, it is trying to access a file insider the directory (`config.providers.aws.param.file.dir= /etc/kafka/`). If we have the secret like `/prod/data/mysql/mypass` then it is expecting a directory structure under the `/etc/kafka`.

```bash
mkdir /etc/kafka/prod/data/mysql/mypass
```
I have raised an issue in their [Github repo](https://github.com/lensesio/secret-provider/issues/29), lets see. But overall, there were no issues with this plugin.  