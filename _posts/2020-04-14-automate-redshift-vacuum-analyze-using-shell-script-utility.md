---
layout: post
title: Automate RedShift Vacuum And Analyze with Script
date: 2020-04-14 06:45:00 +0530
tagline: Automate the RedShift vacuum and analyze using the shell script utility
description: Automate the RedShift vacuum and analyze using the shell script utility
categories:
- RedShift
tags:
- aws
- redshift
- shellscript
- automation
image: "/assets/Automate RedShift Vacuum And Analyze Like a Boss.jpg"
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
---
Vacuum and Analyze process in AWS Redshift is a pain point to everyone, most of us trying to automate with their favorite scripting languge. AWS RedShift is an enterprise data warehouse solution to handle petabyte-scale data for you. AWS also improving its quality by adding a lot more features like Concurrency scaling, Spectrum, Auto WLM, etc. But for a DBA or a RedShift admin its always a headache to vacuum the cluster and do analyze to update the statistics. Since its build on top of the PostgreSQL database. But RedShift will do the Full vacuum without locking the tables. And they can trigger the auto vacuum at any time whenever the cluster load is less. But for a busy Cluster where everyday 200GB+ data will be added and modified some decent amount of data will not get benefit from the native auto vacuum feature. You know your workload, so you have to set a scheduled vacuum for your cluster and even we had such a situation where we need to build some more handy utility for my workload. 

## Vacuum Analyze Utility:

We all know that AWS has an awesome repository for community contributed utilities. We can see a utility for Vacuum as well. But due to some errors and python related dependencies (also this one module is referring modules from other utilities as well). So we wanted to have a utility with the flexibility that we are looking for. And that's why you are here. We developed(replicated) a shell-based vacuum analyze utility which almost converted all the features from the existing utility also some additional features like DRY RUN and etc. Lets see how it works.

You can get the script from my [github repo](https://github.com/BhuviTheDataGuy/RedShift-ToolKit/tree/master/VacuumAnalyzeUtility).

## Script Arguments:

To trigger the vacuum you need to provide three mandatory things. 

1. RedShift Endpoint
2. User Name
3. Database Name

This utility will not support cross database vacuum, it's the PostgreSQL limitation. 
There are some other parameters that will get generated automatically if you didn't pass them as an argument. Please refer to the below table.

| Argument | Details                                                                                    | Default         |
|----------|--------------------------------------------------------------------------------------------|-----------------|
| -h       | RedShift Endpoint                                                                          |                 |
| -u       | User name (super admin user)                                                               |                 |
| -P       | password for the redshift user                                                             | use pgpass file |
| -p       | RedShift Port                                                                              | 5439            |
| -d       | Database name                                                                              |                 |
| -s       | Schema name to vacuum/analyze, for multiple schemas then use comma (eg: 'schema1,schema2') | ALL             |
| -t       | Table name to vacuum/analyze, for multiple tables then use comma (eg: 'table1,table2')     | ALL             |
| -b       | Blacklisted tables, these tables will be ignored from the vacuum/analyze                   | Nothing         |
| -k       | Blacklisted schemas, these schemas will be ignored from the vacuum/analyze                 | Nothing         |
| -w       | WLM slot count to allocate limited memory                                                  | 1               |
| -q       | querygroup for the vacuum/analyze, Default=default (for now I didn't use this in script)   | default         |
| -a       | Perform analyze or not [Binary value, if 1 then Perform 0 means don't Perform]             | 1               |
| -r       | set analyze_threshold_percent                                                              | 5               |
| -v       | Perform vacuum or not [Binary value, if 1 then Perform 0 means don't Perform]              | 1               |
| -o       | vacuum options [FULL, SORT ONLY, DELETE ONLY, REINDEX ]                                    | SORT ONLY       |
| -c       | vacuum threshold percentage                                                                | 80              |
| -x       | Filter the tables based on unsorted rows from svv_table_info                               | 10              |
| -f       | Filter the tables based on stats_off from svv_table_info                                   | 10              |
| -z       | DRY RUN - just print the vacuum and analyze queries on the screen [1 Yes, 0 No]            | 0               |

## Installation:

For this, you just need `psql client` only, no need to install any other tools/software.

## Example Commands:

Run vacuum and Analyze on all the tables.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev 
{% endhighlight %}
Run vacuum and Analyze on the schema sc1, sc2.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -s 'sc1,sc2'
{% endhighlight %}
Run vacuum FULL on all the tables in all the schema except the schema sc1. But don't want Analyze
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -k sc1 -o FULL -a 0 -v 1
or
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -k sc1 -o FULL -a 0
{% endhighlight %}
Run Analyze only on all the tables except the tables tb1,tbl3.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -b 'tbl1,tbl3' -a 1 -v 0
or 
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -b 'tbl1,tbl3' -v 0
{% endhighlight %}
Use a password on the command line.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -P bhuvipassword
{% endhighlight %}
Run vacuum and analyze on the tables where unsorted rows are greater than 10%.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -v 1 -a 1 -x 10
or
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -x 10
{% endhighlight %}
Run the Analyze on all the tables in schema sc1 where stats_off is greater than 5.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -v 0 -a 1 -f 5
{% endhighlight %}
Run the vacuum only on the table tbl1 which is in the schema sc1 with the Vacuum threshold 90%.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -s sc1 -t tbl1 -a 0 -c 90
{% endhighlight %}
Run analyze only the schema sc1 but set the [analyze_threshold_percent=0.01](https://docs.aws.amazon.com/redshift/latest/dg/r_analyze_threshold_percent.html)
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -s sc1 -t tbl1 -a 1 -v 0 -r 0.01
{% endhighlight %}
Do a dry run (generate SQL queries) for analyze all the tables on the schema sc2.
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -s sc2 -z 1
{% endhighlight %}
Do a dry run (generate SQL queries) for both vacuum and analyze for the table tbl3 on all the schema. 
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -t tbl3 -z 1
{% endhighlight %}

## Schedule different vacuum options based on the day

We'll not full the Vacuum full on daily basis, so If you want to run vacumm only on Sunday and do vacuum `SORT ONLY` on the other day's without creating a new cron job you can handle this from the script. 

Just remove this piece of the code.
{% highlight sh%}
if [[ $vacuumoption == 'unset' ]]
	then vacuumoption='SORT ONLY'
else
	vacuumoption=$vacuumoption
fi
{% endhighlight %}
And add this lines.
{% highlight sh%}
## Eg: run vacuum FULL on Sunday and SORT ONLY on other days
if [[ `date '+%a'` == 'Sun' ]]
	then  vacuumoption='FULL'
else 
	vacuumoption="SORT ONLY"
fi
{% endhighlight %}

## Sample output:

**For vacumm and Analyze:**
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -b tbl1 -k sc1 -a 1  -v 1 -x 0 -f 0    
2020-04-13 18:35:25 Starting the process.....
2020-04-13 18:35:25 Validating the Host,User,Database arguments
2020-04-13 18:35:25 Perfect all the mandatory arguments are set
2020-04-13 18:35:25 Validating the Vacuum and Analyze arguments
2020-04-13 18:35:25 Vacuum Arguemnt is 1
2020-04-13 18:35:25 Analyze Arguemnt is 1
2020-04-13 18:35:25 Password will be taken from pgpass
2020-04-13 18:35:25 Getting the list of schema
2020-04-13 18:35:26 Getting the list of Tables
2020-04-13 18:35:26 Setting the other arguments
2020-04-13 18:35:27 Vacuum is Starting now, stay tune !!!
2020-04-13 18:35:28 Vacuum done
2020-04-13 18:35:29 Analyze is Starting now, Please wait...
2020-04-13 18:35:29 Analyze done
{% endhighlight %}

**For Dry Run:**
{% highlight sh%}
./vacuum-analyze-utility.sh -h endpoint -u bhuvi -d dev -s sc3 -a 1  -v 1 -x 80 -f 0 -z 1
2020-04-13 18:33:53 Starting the process.....
2020-04-13 18:33:53 Validating the Host,User,Database arguments
2020-04-13 18:33:53 Perfect all the mandatory arguments are set
2020-04-13 18:33:53 Validating the Vacuum and Analyze arguments
2020-04-13 18:33:53 Vacuum Arguemnt is 1
2020-04-13 18:33:53 Analyze Arguemnt is 1
2020-04-13 18:33:53 Password will be taken from pgpass
2020-04-13 18:33:53 Getting the list of schema
2020-04-13 18:33:54 Getting the list of Tables
2020-04-13 18:33:54 Setting the other arguments

DRY RUN for vacuum
------------------
vacuum SORT ONLY public.nyc_data to 80 percent;
vacuum SORT ONLY sc3.tbl3 to 80 percent;

DRY RUN for Analyze
-------------------
analyze public.nyc_data;
analyze sc3.tbl3;
{% endhighlight %}

## Conclusion:

If you found any issues or looking for a feature please feel free to open an issue on the github page, also if you want to contribute for this utility please comment below. 

## [Click here to get the Code ](https://github.com/BhuviTheDataGuy/RedShift-ToolKit/tree/master/VacuumAnalyzeUtility)