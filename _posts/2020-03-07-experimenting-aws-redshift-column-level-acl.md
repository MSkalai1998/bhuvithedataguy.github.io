---
title: Experimenting AWS RedShift Column Level ACL
date: 2020-03-07 17:20:00 +0530
description: 'Test and explore all the fuctions of RedShift column level acces control'
categories:
- RedShift
tags:
- aws
- redshift
- security
image: "/assets/Experimenting AWS RedShift Column Level ACL.jpg"

---
Good news for the RedShift customers now we can `GRANT` column-level permissions on the tables. It's not only limited to tables, but we can also grant on views and materialized views as well. When the Lake formation was announced, this feature was a part of it. But unfortunately, we need to use Redshift Spectrum to achieve this. The wait is over now. Redshift natively supports the column level restrictions. Im experimenting and walk through this feature and test all the statements mentioned in the Redshift documentation.

## Import a sample table:
For our experiment, we need sample data. So I have download a `.csv` file from [mockaroo](https://mockaroo.com/) and then uploaded the CSV file into my S3 bucket.


{% highlight sql %}
create table test_data (
    id INT,
    first_name VARCHAR(500),
    last_name VARCHAR(500),
    email VARCHAR(500),
    gender VARCHAR(500),
    ip_address VARCHAR(200)
);

COPY test_data  from 's3://my-data-lake/mock_data.csv' iam_role 'arn:aws:iam::1111111111:role/Copying-S3to-RS' CSV;
{% endhighlight %}

I have executed the copy command for multiple times to make my table as some decent amount of rows. 

## Notes about column level ACL:

* You can control the column level access only for `SELECT` and `UPDATE`. 
* Once you assigned some column level restriction, then that user should specifically mention the column names in the query. `Select *` will not work.
* You can control the table, view and materialized views.
* If you want to give both select and update to a user, then just use `GRANT ALL (column names)` will give both access to those columns.
* Table Owner and Superusers can grant the column ACL.

## Experiments:
Before we start our experiment, we can create a user for this.
{% highlight sql %}
create user rs_expriment password 'Mypass123';
{% endhighlight %}
### #1 Grant select only access:
Let's grant select access for a few columns and see how the user can access it in different ways.

{% highlight sql %}
grant select (id, first_name, last_name) on test_data to rs_expriment;

set session authorization rs_expriment;
SET

select id, first_name, last_name from test_data limit 10;
 id | first_name | last_name
----+------------+-----------
  1 | Bernie     | Hull
  3 | Carmina    | Cahill
  5 | Leanora    | Boribal
  7 | Eal        | Crocetto
  9 | Gaylor     | Dugmore
 11 | Pren       | Stenhouse
 13 | Jonie      | Sloegrave
 15 | Heinrik    | Cremen
 17 | Lauri      | Fraser
 19 | Nicolina   | Edwards
{% endhighlight %}

Lets query other column and all columns.

{% highlight sql %}
select id, first_name, last_name, email from test_data limit 10;
ERROR:  permission denied for relation test_data

select * from test_data limit 10;
ERROR:  permission denied for relation test_data
{% endhighlight %}

### #2 Grant UPDATE access

{% highlight sql %}
grant update (first_name) on test_data to rs_expriment;
GRANT

set session authorization rs_expriment;
SET

update test_data set first_name='test_name'  where first_name='Bernie';
UPDATE 13
{% endhighlight %}

Now test with other columns.

{% highlight sql %}
update test_data set last_name='test_name'  where first_name='Bernie';
ERROR:  permission denied for relation test_data

delete from test_data where first_name='test_name';
ERROR:  permission denied for relation test_data
{% endhighlight %}

### #3 select + update togehter

{% highlight sql %}
update test_data set first_name=last_name  where first_name='test_name';
UPDATE 13

update test_data set first_name=email  where first_name='test_name';
ERROR:  permission denied for relation test_data
{% endhighlight %}

### #4 Test the statements from the RedShift Doc
> If a user has a table-level privilege on a table, then granting the same privilege at the column level has no effect.

Anyhow this clearly explains the logic. So we can skip this. 

> If a user has a table-level privilege on a table, then revoking the same privilege for one or more columns of the table returns an error. Instead, revoke the privilege at the table level.

{% highlight sql %}
create user table_full_select password 'MyPass123';
CREATE USER

grant select on test_data to table_full_select;
GRANT

revoke select(id,first_name) on test_data from table_full_select;

ERROR:  Cannot revoke SELECT privilege on test_data.id from user table_full_select as the grantee holds this privilege at the relation level. Revoke the relation level privilege instead.
{% endhighlight %}

> If a user has a column-level privilege, then granting the same privilege at the table level returns an error.

{% highlight sql %}
create user column_select password 'MyPass123';
CREATE USER

grant select(id,first_name) on test_data to column_select;
GRANT

grant select on test_data to column_select;
ERROR:  No privileges were granted. Some grantees hold column privileges on relation test_data. Check for and revoke column level privileges for these grantees on all relations referenced in this Grant statement before granting table privileges to them.
{% endhighlight %}

> If a user has a column-level privilege, then revoking the same privilege at the table level revokes both column and table privileges for all columns on the table.

Note: If you want to revoke the select/update from a column level privilege user, then if you use just `revoke select on` or `revoke update on` will revoke the access. You can use this syntax for revoking access on table level/column level privilege users.

{% highlight sql %}
create user column_select password 'MyPass123';
CREATE USER

grant select(id,first_name) on test_data to column_select;
GRANT

revoke select on table test_data from column_select;
REVOKE
{% endhighlight %}

> You can't grant column-level privileges on late-binding views.

{% highlight sql %}
create view v_test_data as (select * from public.test_data)WITH NO SCHEMA BINDING;
CREATE VIEW
{% endhighlight %}

Try the grant table level access:
{% highlight sql %}
grant select on v_test_data to column_select;
GRANT
{% endhighlight %}
Try the same on late-binding view:
{% highlight sql %}
grant select(id,first_name) on v_test_data to column_select;
ERROR:  column "id" of relation "v_test_data" does not exist.
{% endhighlight %}

> You must have table-level SELECT privilege on the base tables to create a materialized view. Even if you have column-level privileges on specific columns, you can't create a materialized view on only those columns. 

{% highlight sql %}
create user mv_user password 'MyPass123';
CREATE USER

grant select(id,first_name) on test_data to mv_user;
GRANT

create schema acl;
CREATE SCHEMA

grant all on schema acl to mv_user;
GRANT

set session authorization mv_user;
SET

CREATE MATERIALIZED VIEW acl.mv_testdata as (select id, first_name from public.test_data);
CREATE MATERIALIZED VIEW

select * from acl.mv_testdata limit 2;
ERROR:  permission denied for materialized view base relation test_data.
{% endhighlight %}

### oh ho!!! What is this? 
Because by default you have full access on public schema for all the users. Thats why its created. 

### Update: 
Thanks AWS Support team for clarifying this. 
Lets see what happen if have your base table on the different schema? 

{% highlight sql %}
create schema bhuvi;
CREATE

create table bhuvi.test_data (
	id INT,
	first_name VARCHAR(500),
	last_name VARCHAR(500),
	email VARCHAR(500),
	gender VARCHAR(500),
	ip_address VARCHAR(200)
);
insert into bhuvi.test_data (id, first_name, last_name, email, gender, ip_address) values (1, 'Arden', 'Connichie', 'aconnichie0@nifty.com', 'Female', '66.179.130.47');
insert into bhuvi.test_data (id, first_name, last_name, email, gender, ip_address) values (2, 'Elsie', 'Fryatt', 'efryatt1@jugem.jp', 'Female', '91.228.196.151');
insert into bhuvi.test_data (id, first_name, last_name, email, gender, ip_address) values (3, 'Corette', 'Tomasz', 'ctomasz2@miitbeian.gov.cn', 'Female', '134.169.27.141');
insert into bhuvi.test_data (id, first_name, last_name, email, gender, ip_address) values (4, 'Margarita', 'Moulden', 'mmoulden3@google.com', 'Female', '248.63.226.96');
insert into bhuvi.test_data (id, first_name, last_name, email, gender, ip_address) values (5, 'Eran', 'McCoveney', 'emccoveney4@redcross.org', 'Female', '19.137.118.110');

grant select(id,first_name) on bhuvi.test_data to mv_user;
GRANT

set session authorization mv_user;
SET

CREATE MATERIALIZED VIEW bhuvi.mv_testdata as (select id, first_name from bhuvi.test_data);
ERROR:  permission denied for schema bhuvi
{% endhighlight %}

## Conclusion:
Its not a big deal to work with column level ACL. But its worth to test every small feature. So you have better visibility about the feature and find the bugs like what we found above. I'll update this blog once the AWS team confirms this as a bug or not. 