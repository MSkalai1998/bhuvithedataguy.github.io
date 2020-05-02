---
layout: post
date: 2020-04-26 20:00:00 +0530
tagline: Generate RedShift copy command from SCT agent exported to S3 or Snowball with random string folders.  
description: AWS SCT Extraction Agents will help to pull the data from the various sources. Generate RedShift copy command from SCT agent exported to S3 or Snowball with random string folders. 
categories:
- RedShift
tags:
- aws
- redshift
- shellscript
- automation
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
---
AWS SCT Extraction Agents will help to pull the data from the various data sources and push it into the targets. S3 and Snowball also can be a target for SCT Agent. But if we run the SCT Agent with multiple tasks and multiple agents they will export the data into S3 or Snowball with some string folder structure. If you want to push it into the RedShift then it is very difficult to import from the complex random string(UUID kind of) path. Recently we had an experience with this where we have to import around 20TB+ data from Snowball(later exported to S3) to RedShift. In this blog, Im going to share my experience and script to generate RedShift copy command from SCT agent exported to S3 or Snowball with random string folders. 
![](/assets/RedShift Copy Command From SCT Agent Export In S3.png)
## SCT Agent Terminology:

Before explaining the solution, let's understand how the SCT agent works and its terminology.

1. Once we created a migration task, it'll split the data export between all the extraction agents.
2. One table's data will be exported in multiple chunks. This chunk is based on the Virtual Partition. 
3. The Virtual partition will be considered as Subtasks for the export. 

So the export folder structure will be formed based on the above terminology.

* Top-level folder - The S3 prefix that you are going give while creating the task.
* 2nd level - Extraction Agent ID (Each agent has a name as well as a unique ID).
* 3rd level - The subtask ID. Each virtual partition will be exported in a subtask. 
* 4th level - The main task ID. The job you created to export, it'll get an ID.

**Lets see an example**

* **S3 Prefix** - april-my-large-table/
* **Extraction Agent ID** (you can have multiple agents, but here I'll select the agent 1's ID) - e03e92407fcf4ede86e1f4630409c48c 
* **Subtask ID** - 19f6ce57ec254053ab5eb72245dc2641
* **Task ID** - 4c0233f054c840359d93bdae8c175bf8

Now if you start exporting the data then it'll use the following folder structure.

`s3://bucketname/s3-prefix/agent-id/subtask-id/task-id/`

Lets compare this with our example.

`s3://migration-bucket/april-my-large-table/e03e92407fcf4ede86e1f4630409c48c/19f6ce57ec254053ab5eb72245dc2641/4c0233f054c840359d93bdae8c175bf8/`

## S3 export Terminology:

We understand the folder/directory structure, now let's see what kind of files we'll get inside these folders.

* **unit directory** - Your actual data in CSV format will be pushed here. You'll get up to 8 files per unit folder. If your data size is huge then you'll get multiple unit folder like `unit_1, unit_2, unit_n`.
* One good thing with this SCT is it'll generate the COPY command for the RedShift using Manifest file. So inside your unit directory if you have 8 files, then there is a file called `unit.manifest` which is a manifest file that contains all the 8 files full path. Along with this, you'll get the SQL file that already refers to the manifest file.  So using this SQL we can load all the 8 CSV files. 
* **Statistics file** - This is a JSON file that contains the information about the Taskid, agent ID, source, and target table information and the where condition used to export this data(I virtual partition or chunk). This statistics file will have the name that is the same as your subtask ID.

Refer to the below image which shows the complete illustration of the SCT Agent directory structure. And if you notice the subtask ID and the statistic file's name, both are the same. 

![RedShift Copy Command From SCT Agent Export In S3](/assets/RedShift Copy Command From SCT Agent Export In S3 snap.jpg "RedShift Copy Command From SCT Agent Export In S3")
![RedShift Copy Command From SCT Agent Export In S3](/assets/RedShift Copy Command From SCT Agent Export In S3 snap1.jpg "RedShift Copy Command From SCT Agent Export In S3")

## Challenges:

Here can notice one thing, it's already in good shape to load the CSV files to RedShift. Just get all the SQL files and run them, then what is the challenge here? 
* If you extract the SQL file, there we have an option for using Access and Secret keys. But we have to manually fill these values.
* I mean download all the SQL files(all are having the same name, so while downloading rename them). And edit them with the right Access and Secret key values.
* One SQL command refers to one manifest file that means a maximum of 8 files is there. If you have a pretty large cluster that has more than 8 slices then the performance will be slow. 
* If you have a Terabyte of data for a single table, you'll get so many SQL files. Then you have to write a script to loop all these COPY SQL files one by one which may take a longer time. 

## Optimized Approach: 

We can generate a custom SQL file for COPY command that refers to all the CSV files from the multiple subfolders. So in one SQL command, we can load all of our CSV files to RedShift. So how do we get that? 

1. Get the list of all the `unit.manifest` files.
2. Download the manifest files to your system.
3. Extract the CSV files location from all the manifest files. For this Im going to use `jq` a command-line JSON parser for Linux. So just extract the `entries[]` array from the `unit.manifest` file.
4. Create a new manifest file that contains all the CSV file locations that are extracted from the previous step. And upload this to the new S3 path.
5. Generate a SQL command that refers to the newly generated manifest.
6. If you have many tables instead of repeating this step for all the tables, pass the table names, and the S3 prefix in a CSV file, let the script to the complete work for you.

## Remember how did you export? 

1. From SCT we can create a local task to export a single table, so the S3 prefix contains the data only for that particular table. 
2. Or you can upload a CSV file to SCT that contains a bunch of tables list. In this case, your S3 prefix location contains all the tables files. But luckily one subtask will export a table's data. So it'll not mix multiple tables data in one subtask.

And make sure you install `jq`.
{% highlight sh%}
apt install jq
yum install jq
{% endhighlight %}

**Replace the following things in my script**

* **migration-bucket** - your bucket name
* **your_schema** - RedShift schema name
* **1231231231** - AWS Account ID for the RedShift IAM role
* **s3-role** - RedShift IAM role name to access S3
* **ap-south-1** - s3 bucket region
* **merged_manifest** - New s3 prefix where the new Manifest file will get uploaded

## Script to generate SQL for individually exported tables:

Prepare a CSV file that contains the list of table names and their s3 Prefix. 

**Eg Filename: source_generate_copy**
{% highlight sh%}
table1,apr-04-table1
table2,apr-04-table2
table3,apr-04-table3
{% endhighlight %}

**Create necessary folders**
{% highlight sh%}
mkdir /root/copy-sql  
mkdir /root/manifest-list  
mkdir /root/manifest_files
{% endhighlight %}

**Download the Manifest**

I want to download 20 files parallelly, so I used `&` and `((++count % 20 == 0 )) && wait`. If you want to download more then use a higher number. But it better to keep less than 100. 
{% highlight sh%}
count=0
while read -r s_table
do
table=$(echo $s_table | awk -F',' '{print $1}')
s3_loc=$(echo $s_table | awk -F',' '{print $2}')
aws s3 ls --human-readable --recursive s3://migration-bucket/$s3_loc/ | grep "unit.manifest" | awk -F ' ' '{print $5}' > /root/manifest-list/$table.manifest.list      
mkdir -p /root/manifest_files/$table/
file_num=0
while read -r r_manifest
do
aws s3 cp s3://migration-bucket/$r_manifest /root/manifest_files/$table/unit.manifest.$file_num &
file_num=$(( $file_num + 1 ))
((++count % 20 == 0 )) && wait
done < /root/manifest-list/$table.manifest.list
done < source_generate_copy
{% endhighlight %}

**Merge Manifest into one**
{% highlight sh%}
while read -r s_table
do
table=$(echo $s_table | awk -F',' '{print $1}')
files=$(ls /root/manifest_files/$table)
for file in $files
do
echo $file
cat /root/manifest_files/$table/$file | jq '.entries[]'  >> /root/manifest_files/$table/unit.merge
done
cat /root/manifest_files/$table/unit.merge | jq -s '' > /root/manifest_files/$table/$table.manifest
sed -i '1c\{"entries" : ['  /root/manifest_files/$table/$table.manifest
sed -i -e '$a\}'  /root/manifest_files/$table/$table.manifest
done < source_generate_copy
{% endhighlight %}

**Upload Manifest**
{% highlight sh%}
while read -r s_table
do
table=$(echo $s_table | awk -F',' '{print $1}')
aws s3 cp /root/manifest_files/$table/$table.manifest s3://migration-bucket/merged_manifest/
done < source_generate_copy
{% endhighlight %}

**Generate COPY**

Change the options as per your need. 
{% highlight sh%}
while read -r s_table
do
table=$(echo $s_table | awk -F',' '{print $1}')
echo "COPY your_schema.$table from 's3://migration-bucket/merged_manifest/$table.manifest' MANIFEST iam_role 'arn:aws:iam::1231231231:role/s3-role' REGION 'ap-south-1' REMOVEQUOTES IGNOREHEADER 1 ESCAPE DATEFORMAT 'auto' TIMEFORMAT 'auto' GZIP DELIMITER '|' ACCEPTINVCHARS '?' COMPUPDATE FALSE STATUPDATE FALSE MAXERROR 0 BLANKSASNULL EMPTYASNULL  EXPLICIT_IDS"  > /root/copy-sql/copy-$table.sql
done < source_generate_copy
{% endhighlight %}

All tables COPY command will be generated and located on `/root/copy-sql/` location.

I have written about how to generate the SQL command for bulk exported tables in **[Part 2](https://thedataguy.in/redshift-copy-script-from-sct-agent-export-s3-part1/)**. 

