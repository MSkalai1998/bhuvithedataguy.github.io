---
title: RedShift Reconstructing SQL from STL_QUERYTEXT
date: 2020-03-18 07:37:00 +0530
description: 'Reconstruct all SQL queries from STL_QUERYTEXT with LISTAGG and partition by'
categories:
- RedShift
tags:
- aws
- redshift
- sql
image: "/assets/RedShift Reconstructing SQL from STL_QUERYTEXT.jpg"

---
If you are managing the RedShift clusters then **STL_QUERY** and **STL_QUERYTEXT** tables are not new to you. STL_Query can't hold the complete SQL query instead we can use STL_QueryText to read the complete query. But there is a challenge, we can't read that table as it is. Since your queries are saved in multiple rows. So we need to combine all these rows into a single row with LISTAGG function which is well documented [here](https://docs.aws.amazon.com/redshift/latest/dg/r_STL_QUERYTEXT.html).

## What is the real challenge? 

From the AWS documentation example, they are showing to reconstruct the SQL for a single query. But in a production system, you may have 50k+ queries where the LISTAGG function will through some errors related to its limitation. Let's simulate this. I ran the below query on my production RedShift Cluster.
{% highlight sql %}
SELECT   Listagg( 
         CASE 
                  WHEN Len(Rtrim(text)) = 0 THEN text 
                  ELSE Rtrim(text) 
         END, '') within GROUP (ORDER BY SEQUENCE) AS text
FROM     stl_querytext limit 10;

ERROR:  Result size exceeds LISTAGG limit
DETAIL:
  -----------------------------------------------
  error:  Result size exceeds LISTAGG limit
  code:      8001
  context:   LISTAGG limit: 65535
  query:     180132
  location:  0.cpp:228
  process:   query0_78_180132 [pid=23735]
  -----------------------------------------------
{% endhighlight %}
## Why this error? 

If you read the LISTAGG function details, If the result set is larger than the maximum VARCHAR size (64K – 1, or 65535), then LISTAGG returns the error. Here I have more than 50k+ rows. So the overall characters will not fit in the result set.  

## What is the solution?:

If we run the reconstruct query one by one instead of all then the resultset will fit into the LISTAGG's limitation. Lets say one of my queries split into 2 rows in the STL_QUERYTEXT table.

{% highlight sql %}
SELECT text 
FROM   stl_querytext 
WHERE  query = 97729 
ORDER  BY SEQUENCE; 
{% endhighlight %}

![](/assets/RedShift Reconstructing SQL from STL_QUERYTEXT1.jpg)

First, process these two rows and then process another query and then the next one.

## How? 

We have `userid`,`pid`,`xid`,`query` columns are common between these two rows. So partition your resultset by query wise. Then do the LISTAGG. 
{% highlight sql %}
LISTAGG(
      CASE
        WHEN LEN(RTRIM(text)) = 0 THEN text
        ELSE RTRIM(text)
      END,
      ''
    ) within group (
      order by
        sequence
    ) over (PARTITION by userid, xid, pid, query) 
{% endhighlight %}

## Replace \n and \r characters:
If you are going to run this query from psql client most of the queries starts with `\r\nselect * from mytbl` or any GUI based tools will return `\n` characters. 
* \r actually a carriage return
* \n is not a new line its a sting. 
So we have to replace twice.
{% highlight sql %}
replace(replace(
    LISTAGG(
      CASE
        WHEN LEN(RTRIM(text)) = 0 THEN text
        ELSE RTRIM(text)
      END,
      ''
    ) within group (
      order by
        sequence
    ) over (PARTITION by userid, xid, pid, query),
    '\r','' 
  ), '\\n', '')
{% endhighlight %}
## Final View:

For east management, Im going to create a view on top of the STL_QUERYTEXT table to return the reconstructed SQL for all the queries.
{% highlight sql %}
CREATE VIEW recon_stl_querytext 
AS 
  (SELECT DISTINCT userid, 
                   xid, 
                   pid, 
                   query, 
                   Replace(Replace(Listagg(CASE 
                                             WHEN Len(Rtrim(text)) = 0 THEN text 
                                             ELSE Rtrim(text) 
                                           END, '') 
                                     within GROUP ( ORDER BY SEQUENCE ) over ( 
                                       PARTITION BY userid, xid, pid, query), 
                           '\r' 
                           , ''), '\\n', '') 
                   AS text 
   FROM   stl_querytext 
   ORDER  BY 1, 
             2, 
             3, 
             4); 
{% endhighlight %}  

Now you can view the proper data from this view.
{% highlight sql %}
SELECT * 
FROM   recon_stl_querytext 
WHERE  query = 97729; 
{% endhighlight %}

![](/assets/RedShift Reconstructing SQL from STL_QUERYTEXT2.jpg)

I hope this helps you to get the overall consolidated view on the query history. If you are interested in how to [Audit RedShift Historical Queries With pgbadger please click here](https://medium.com/searce/audit-redshift-historical-queries-with-pgbadger-619f7f43fbd0).
