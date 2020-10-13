---
layout: post
date: 2020-10-13 08:05:00 +0530
title: CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role
description: Enabling private IP on GCP CloudSQL failed with an error message set Service Networking service account as servicenetworking.serviceAgent role on consumer project.
categories:
- GCP
tags:
- gcp
- cloudsql
- networking
social:
  name: Bhuvanesh
  links:
    - https://twitter.com/BhuviTheDataGuy
    - https://www.linkedin.com/in/rbhuvanesh
    - https://github.com/BhuviTheDataGuy
image: "/assets/gcp/CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role.jpg"
---
**Enabling private IP on GCP CloudSQL failed with an error message set Service Networking service account as servicenetworking.serviceAgent role on consumer project.**

GCP's CloudSQL and its private IP communication is a little bit tricky while comparing with the other CloudSQL vendors like AWS RDS, Azure SQL for MySQL and PostgreSQL. CloudSQL will be launched inside the GCP's VPC. But if we need a private communication between the CloudSQL and our VPC, then we need to enable the Private IP option.

## How Private Communication works with GCP:

When we enable the private IP on CloudSQL, GCP picks an IP address from the Private IP address allocation(from VPC console you can allocate a different IP ranges) and start establishing a peering between our VPC and Google's CloudSQL VPC and the IP address will be allocated to the CloudSQL. By default, all the IP addresses in our VPC will be allowed to access the CloudSQL and we can't control allow or block private IPs. 

## What I feel bad about this:

* Once we enabled the private IP, we can't disable them.
* If you want to assign the private IP to an existing instance to a Shared VPC is not possible. 
* Even though if you are a project editor, still you need `Computer Network Admin` permission to enable the private IP on the CloudSQL.
* And Service Networking API needs to be enabled.

## Private IP with Shared VPC:

Recently I was working on a project where we have a project dedicated to  Network and subnets. And this VPC shared with a few other projects/ Then we have other projects where all of our resources will be launched inside the Shared VPC. One of the consumer projects, when I tried to launch a CloudSQL instance and if try to enable, it was throwing the following error message.
```bash
set Service Networking service account as servicenetworking.serviceAgent role on consumer project.
```
We already have 3 CloudSQL instances with Private IP enabled(Before the Shard VPC configured). But this time something went wrong. I had project owner permission on both the host and the consumer project, but still, it didn't work. And when I read the error message, its clearly says its a permission issue. When I google it, I found many folks had the same issue and they suggested Disable and Enable the `Service Networking API` will solve the issue. 

## Private Secure Connection - Error Loading Data:

Then I wanted to check any changes on the VPC side. I checked the VPC's private secure connection. And the Private allocation table is showing the allocated private IP address. Then I switched to `Private connection to services` tab, it was showing Error Loading Data. This made us think that something wrong with the Host project that needs to be fixed. 
{% include lazyload.html image_src="/assets/gcp/CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role1.png" image_alt="CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role" image_title="CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role" %}

## Disable and Enable the Service Networking API:

But there is a glitch, all the resources that are launched by this API may be destroyed if we disable the Service Networking API. And the error message says enable it on the Consumer account, not the host project. So we took all the backups and tried the workaround. Unfortunately, it didn't work for us. 

## servicenetworking.serviceAgent role:

From the error message, we understood that we have to assign this permission to the service account(but it won't support assigning it to a user).  And Im not able to see any service account that ends the suffix as `service-networking.iam.gserviceaccount.com` on the Host and Consumer project. After a few attempts, we ended up with manually assign the `servicenetworking.serviceAgent` to the project using IAM Bind Policy.

## Bind the IAM Policy:

As a last try, I wanted to assign the role to the project. For this, we need to know the account ID of the Host project. You can get it from IAM --> Settings.
```bash
gcloud projects add-iam-policy-binding YOUR_HOST_PROJECT_NAME \
  --member=serviceAccount:service-HOST_PROJECT_ACCOUNT_ID@service-networking.iam.gserviceaccount.com \
  --role=roles/servicenetworking.serviceAgent
```
After this, all the project owner,editors and network admin's and some Service accounts got updated with the `servicenetworking.serviceAgent` role.
```bash
Updated IAM policy for project [mgmt].
bindings:
- members:
  - serviceAccount:instance-sa@XXXXXX.iam.gserviceaccount.com
  - serviceAccount:metrics-loader@project.iam.gserviceaccount.com
  - serviceAccount:xxxx-production@XXXXXX.iam.gserviceaccount.com
  - user:bhuvanesh@domain.com
  role: roles/compute.networkAdmin
```
Now when I checked the private connections to the services, CloudSQL and Service Networking APIs are enabled. And for us, nothing destroyed. Everything was up and running.
{% include lazyload.html image_src="/assets/gcp/CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role2.png" image_alt="CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role" image_title="CloudSQL Shared VPC Private IP And servicenetworking.serviceAgent role" %}