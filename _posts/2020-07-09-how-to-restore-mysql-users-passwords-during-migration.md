---
title: How To Restore MySQL Users And Passwords During Migration
date: 2020-07-09 15:10:00 +0530
description: Restore mysql users with the exact password from one server to another server during migration. You can use this instead of using pt-show-grants and mysqlpump.
categories:
- MySQL
tags:
- mysql
- security
image: "/assets/mysql-restore-users/How To Restore MySQL Users And Passwords During Migration.jpg"
---
In any MySQL replication or migration, generally, we'll skip the `mysql` database from the dump file. Since it has the current server's settings, users, etc. So we should not create a mess on the target server with this source mysql database dump. But one downside of this approach is we'll not get the users on the target MySQL server. Maybe you tried some approaches that are mentioned [here](https://stackoverflow.com/questions/597732/backup-mysql-users). But whenever I don't the credentials list during the migration and don't have time to explore/install any tools like[pt-show-grants](https://www.percona.com/doc/percona-toolkit/LATEST/pt-show-grants.html), then I'll use this trick.

> **DISCLAIMER** You should try this approach when you don't any user credentials on the source DB. Take a backup of the MySQL database on the target DB side. Do this with your DBA's help. And it is tested in MySQL 5.6 and 5.7

## Step #1: Get the grant:

```bash
-- Get grants
mysql -h SOURCE_DB_IP_ADDRES -u root -p'PASSWORD' --skip-column-names -A -e"SELECT CONCAT('SHOW GRANTS FOR ''',user,'''@''',host,''';') FROM mysql.user WHERE user<>''" | mysql -h IP_ADDRES -u root -p'PASSWORD' --skip-column-names -A | sed 's/$/;/g' > user_grants.sql

-- clean up the password field
-- If the source is RDS
sed -i 's/IDENTIFIED BY PASSWORD <secret>//g' user_grants.sql

-- If the source is VM/CloudSQL/or whatever
sed -i "s/IDENTIFIED[^']*'[^']*//" user_grants.sql
```

## Step #2: Generte create user statement:

```bash
-- MySQL 5.6
mysql -h source_DB -u root -p'password' --skip-column-names -A mysql -e "SELECT Concat('create user \'', user, '\'@\'', host, '\' IDENTIFIED WITH \'mysql_native_password\' AS \'', password,'\';') FROM   mysql.user where user not in ('mysql.session','mysql.sys','debian-sys-maint','root');" > create_user.sql

-- MySQL 5.7
mysql -h source_DB -u root -p'password' --skip-column-names -A mysql -e "SELECT Concat('create user \'', user, '\'@\'', host, '\' IDENTIFIED WITH \'mysql_native_password\' AS \'', authentication_string,'\';') FROM   mysql.user where user not in ('mysql.session','mysql.sys','debian-sys-maint','root');" > create_user.sql
```
# Step #3: Restore the users:

```bash
mysql -u target_db_ip -u root -p myqsl < create_user.sql
mysql -u target_db_ip -u root -p myqsl < user_grants.sql
```

## Bouns:
### Credits: [Pankaj](https://www.linkedin.com/in/pankaj-kumar-99164a100/)

If you want to rename a user like change the HOST part, then you can use the following commands.

```bash
mysql -h source_DB -u root -p'password' --skip-column-names -A mysql -e "SELECT Concat('rename user \'',user,'\'@\'',host,'\' to \'',user,'\'@\'','10.20.4.240\'',';') from mysql.user where user not in ('mysql.session','mysql.sys','debian-sys-maint','root','mysql.infoschema');" > rename_user.sql

mysql -u target_db_ip -u root -p myqsl < rename_user.sql

```

Again a friendly reminder, keep your DBA with you and do take a backup of the MySQL database. If required you can still use [pt-show-grants and mysqlpump](https://mysqlstepbystep.com/2018/05/14/backing-up-users-and-privileges-in-mysql/) as well.

