# Overview

This repo contains a collection of AWS [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) templates intended to help you set up common pieces of AWS infrastructure. Each template defines a [stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html), which is a collection of related resources that can be created or deleted as a single unit. Templates are available for creating:
- A secure network inside a [VPC](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html)
- A [bastion host](https://en.wikipedia.org/wiki/Bastion_host) to securely access instances inside the VPC
- A deployment environment using AWS [Elastic Beanstalk](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/Welcome.html)
- A container-based environment using [Amazon Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_GetStarted.html)
- A relational database using [Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)
- An [Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Aurora.html) DB cluster
- [Billing alerts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics), to monitor your costs

The VPC template is a requirement for the others. You can either run the templates/vpn.cfn.yml template by itself prior to using the others, or run any one of the vpn-*.cfn.yml wrapper templates at the top level of this repo to create sets of resources in the same stack.

## Prerequisites

If you haven't already done so you first need to:
- [Create an AWS account](https://aws.amazon.com/blogs/startups/how-to-get-started-on-aws-from-a-dead-standstill/).
- Make sure you're signed into AWS as an [IAM user with admin access](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html). Avoid using the root account!
- [Create an EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair). This is necessary to use the bastion host.
- [Clone this repo](https://help.github.com/articles/cloning-a-repository/) so that you have a local copy of the templates.

## Creating stacks
Use the AWS [CloudFormation Console](https://console.aws.amazon.com/cloudformation/home) to run the templates. Click the "Create Stack" button in the upper left corner of the console, then under "Choose a template", select "Upload a template to Amazon S3" and click "Browse" to find your local fork of this repository and choose the template you want to run.

## The templates

Here’s a description of the resources created by each file in the /templates directory:


### VPC

The vpc.cfn.yml template is a prerequisite for most of the others--you need to either run it first, or run one of the wrapper templates at the top level of the repo, which include it. It creates an [Amazon Virtual Private Cloud](https://aws.amazon.com/vpc/, a virtual data center in which you can securely run AWS resources. It also creates related networking resources:
- 2 Public [subnets](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html)
- 2 Private subnets
- 1 [Internet gateway](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html)
- 1 [NAT gateway](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html)
- 3 [route tables](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Route_Tables.html)
- A bunch of [security groups](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Security.html).

Subnets are isolated network areas--public subnets are visible to the Internet, private ones can only be reached from inside the VPC. If a resource in a private subnet has to communicate externally it has to do so via a NAT Gateway, which acts as a proxy.

The VPC template creates two public and two private subnets, in different [Availability Zones (AZ)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) for redundancy. A subnet is public if it’s associated with an Internet gateway, which allow it to communicate with the Internet

Each subnet has to be associated with a route table, or set of network rules, that define allowed traffic. Route tables operate at the subnet level. The VPC template creates two of them, one for the public subnets, and one for the private.

Security groups act as firewalls at the instance level, to control inbound and outbound traffic. The template creates security groups for an application, load balancer, database, and bastion host. Depending on what other templates you run, not all of them may be used.


### Bastion host

It's preferable not to ssh into instances at all, instead configuring them to send logs to [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) or other services, and managing instantiation, configuration, and termination of instances using devops tools.

If you do need to connect directly to EC2 instances, it's best practice (and for instances in a private subnets, a requirement) to use a bastion host, otherwise known jump box. A bastion host is an instance that is publicly accessible, and which is also has access resources in private subnets, so it can act as a secure go between--you can ssh into the bastion host, and from there into your other instances. When you run the bastion.cfn.yml template, it creates:
- An t2.micro EC2 instance
- An Elastic IP Address
- An Elastic Network Interface

The bastion template is dependent on having previously run the VPC template.

### Elastic Beanstalk

AWS Elastic Beanstalk is a service that lets you define an environment for common application types, and deploy code into it. The Beanstalk template is dependent on the VPC, and optionally can be used with the bastion, RDS, or Aurora templates.

When you run the template it asks for a series of inputs defining your environment. Those with constrained values are:
- A stack type, with allowed values of node, rails, python, python3 or spring.
- An environment name with allowed values  of dev or prod.

It creates:
- A service role
- And Elastic Beanstalk application
- An Elastic Beanstalk environment
- An Auto Scaling Group
- A Load Balancer 


### Fargate

[AWS Fargate](https://aws.amazon.com/fargate/) is part of [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/). It's a managed service for running container-based applications, without having to worry about the underlying servers--sort of like [Lambda](https://aws.amazon.com/lambda/) for containers.

It creates:
- An S3 bucket for the container
- An S3 bucket for CodePipeline artifacts
- A CodePipeline
- A CodePipeline service role
- A CodeBuild project
- A CodeBuild service role
- An Elastic Container Repository (ECR) repository
- An Application Load Balancer (ALB)
- An ALB Route 53 record
- ELB target groups stuff
- A Fargate task definition
- A Fargate service with associated scaling resources


### RDS

[Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) is a service for running relational databases without having to manage the server software, backups, or other maintenance tasks. The RDS service as a whole supports Amazon Aurora, PostgreSQL, MySQL, MariaDB, Oracle, and Microsoft SQL Server; the template included here currently works with PostgreSQL, MySQL, and MariaDB.

It creates:
- A DB instance
- A DB subnet group


### Aurora

Amazon Aurora is a high-performance cloud-optimized relational database, which is compatible with MySQL and PostgreSQL. It’s treated separately than RDS because Aurora has a few unique characteristics.

It creates:
- A DB Cluster
- An Aurora DB instance
- A DB subnet group

### Billing Alerts

If you leave AWS resources running longer than intended, have unexpected traffic levels, or misconfigure or over-provision resources, your bill can climb higher or faster than expected. To avoid surprises we recommend turning on [billing alerts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics), so that you're notified when charges go above preconfigured thresholds. The billing_alert template makes this easier.

Before running it you need to use the AWS console to enable billing alerts:

- Log into the [billing section of the console](https://console.aws.amazon.com/console/home), also reachable by clicking your username on the top right, and selecting My Billing Dashboard.
- Select Preferences from the list of options on the left.
- Check Receive Billing Alerts. Once saved this cannot be disabled.

Now you can run the billing_alert.cfn.yml template. You'll be asked for the threshold (in US dollars) for receiving an alert and the email address the alert should be sent to. If you want to get alerts at more than one threshold, you can run the template multiple times. The template will create:
- A [CloudWatch alarm](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cw-alarm.html)
- An [SNS topic](https://docs.aws.amazon.com/sns/latest/dg/CreateTopic.html)

You can read about more [ways to avoid unexpected charges](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/checklistforunwantedcharges.html).