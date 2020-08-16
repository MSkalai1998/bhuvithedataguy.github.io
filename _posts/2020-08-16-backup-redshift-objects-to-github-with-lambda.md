---
layout: post
date: 2020-08-16 22:00:00 +0530
title: Backup RedShift Objects To GitHub With Lambda
description: Export RedShift schema and objects like Tables, users, schema and other DDL to Github from AWS Lambda. Use PyGithub to export files from AWS Lambda.
categories:
- RedShift
tags:
- aws
- redshift
- lambda
- python
- github
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/redshift/Backup RedShift Objects to GitHub with Lambda.jpg"
---
**Backup RedShift database objects like Schema, Tables, User grants, user groups to GitHub with AWS Lambda. We can use PyGithub library to commit the files from Lambda.**

RedShift is built on top of PostgreSQL. But unfortunately, there are no native dump tools to backup the redshift objects. Of course, `pg_dump` is there, but it'll not dump the structure with Column encode, Distribution style, etc. To solve this problem AWS have an amazing repo [RedShift Utilities](https://github.com/awslabs/amazon-redshift-utils/tree/master/src/AdminViews). We have multiple Admin views to generate the DDL for almost all the objects. Im coming from SQL DBA background and I read a feature from [DBAtools](https://www.sqlservercentral.com/blogs/backup-your-sql-instances-configurations-to-git-with-dbatools-part-1) that will support export SQL Server objects to Github. So I was think to implement such export utilitiy for RedShift as well. And thats why you are here to read about it.

## Prepare the infra:

* Lambda needs to be run on a private subnet. (to ensure the security), so those subnet's IP range must be whitelisted on your RedShift's security group.
* The Lambda subnets should have a NAT gateway to communicate with the GitHub.
* Lambda function's security group should allow the RedShift's Port number, HTTP, HTTPS ports in the outbound rules.
* Its better to use RedShift master user account.
* In this blog, Im using Environment variables but without encrypting any variables. I highly recommend to use KMS or Parameter store to encrypt the RedShift and Github credentials. 
* Lambda basic execution and Lambda VPC execution policies are enough to the Lambda IAM role. If you need you can modify it.
* Create the following admin views in RedShift's Admin schema.
	* [v_get_users_in_group](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_get_users_in_group.sql)
	* [v_generate_user_object_permissions](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_generate_user_object_permissions.sql)
	* [v_generate_schema_ddl](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_generate_schema_ddl.sql)
	* [v_generate_tbl_ddl](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_generate_tbl_ddl.sql)

## Lambda Layer for Github & psycopg2 Library:

[`PyGithub`](https://pypi.org/project/PyGithub/) and [`psycopg2`](https://pypi.org/project/psycopg2/) are not nativly avaiable in AWS Lambda. So we are going to use a Lambda layer to support these libraries. You can simply create a new layer and [`upload this ZIP file`](https://github.com/BhuviTheDataGuy/medium-blog-files/raw/master/thedataguy/redshift-github/pygithub-psycopg2-lambda-layer.zip). Or manually create the ZIP package by [`using this steps`](https://stackoverflow.com/a/63425782/6885516). I used **Python 3.8 runtime**.

## Github Acces Token:

In the Python code, to make the authendication to Github we can use either username and password or access token. I used the access tokens. for this blog. You can follow [`this link`](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) to generate an access token.

## Lambda function:

* Once the Layer and the token is ready then create a new lambda function inside a VPC with Python 3.8 add the layer that you created in the previous step. 
* Then from the Function code --> Actions select the `Upload a zip file` option. **[Download the ZIP file here](https://github.com/BhuviTheDataGuy/medium-blog-files/raw/master/thedataguy/redshift-github/lambda-function.zip)** and upload it. 
* It has many Environment variables. So create the following Env variables in Lambda.
```bash
REDSHIFT_DATABASE
REDSHIFT_USER
REDSHIFT_PASSWD
REDSHIFT_PORT  
REDSHIFT_ENDPOINT
GITHUB_ACCESS_TOKEN
GITHUB_REPO 
GITHUB_PREFIX
```
> I recommend to encrypt it with KMS or use Parameter store. If you use KMS then from the lambda function, you may need to make some chnges on the Env variables section (under the #Env variables line).
Increase the function timeout to 15mins.

## Code flow

This is just a code walkthrough, not the complete code. Hit [this link](https://github.com/BhuviTheDataGuy/medium-blog-files/raw/master/thedataguy/redshift-github/lambda-function.zip) to get the complete code in ZIP file.

### Get the available files from Github:

PyGithub library will create a new file or update an exising file. So we have make sure that if the file is not available already the use `CREATE` attribute else use `UPDATE` attribute. So get all the files from the repo.
```py
# Get all the exsiting files
repo = g.get_user().get_repo(GITHUB_REPO)
all_files = []
contents = repo.get_contents("")
while contents:
    file_content = contents.pop(0)
    if file_content.type == "dir":
        contents.extend(repo.get_contents(file_content.path))
    else:
        file = file_content
        all_files.append(str(file).replace('ContentFile(path="','').replace('")',''))
```
### Execute the Admin views one by one

Get the list of queries from the `queries` directory and then execute it on RedShift. Fetch the results and save them into a file.
```py
query_list = ['gen_schema', 'gen_table', 'gen_usergroup', 'gen_adduser_group', 'gen_object_grants']
for query in query_list:

    # read SQL query
    with open ('queries/' + str(query) + '.sql') as file:
        sqlquery = file.read()
    cur.execute(sqlquery)
    result = cur.fetchall()
    # Process all the rows one by one
    rows = []
    for row in result:
        rows.append(''.join(row).encode().decode('utf-8'))
    
    # Write the results to a file
    with open(path + query + '.sql', 'w') as file:
        for item in rows:
            file.write("%s\n" % item)
```
### Upload files to GitHub

Read the contents from the output files that are generated from the previous step and then upload it to Github. `GITHUB_PREFIX` is the variables used to define the directory inside the GitHub repo to upload the files. Before uploading them check the files are already available. If NO then create them else update them.
```py
# Read the result file
with open(path + query + '.sql', 'r') as file:
    content = file.read()

# Upload to github
git_prefix = GITHUB_PREFIX
git_file = git_prefix + query + '.sql'
if git_file in all_files:
    contents = repo.get_contents(git_file)
    repo.update_file(contents.path, gitcomment, content, contents.sha, branch="master")
    print(git_file + ' UPDATED')
else:
    repo.create_file(git_file, gitcomment, content, branch="master")
    print(git_file + ' CREATED')
```
##Conclusion:

DevOps is chaning the database culture as well. The intension of this blog to keep the RedShift schema or objects on Github and intergrate with your CI/CD pipeline. Consider this blog a kickstart doc and keep improvsing your data pipeline. 