<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Overview](#overview)
- [Prerequisites for this setup](#prerequisites-for-this-setup)
  - [Install AWSCLI and Jq](#install-awscli-and-jq)
- [Quick start: Create AWS CloudTrail](#quick-start-create-aws-cloudtrail)
- [Third party visualized reporting tools](#third-party-visualized-reporting-tools)
- [The CloudTrail and Spunk integration](#the-cloudtrail-and-spunk-integration)
  - [Why use CloudTrail](#why-use-cloudtrail)
  - [Create an IAM user for Splunk integration](#create-an-iam-user-for-splunk-integration)
  - [Subscribe Simple Queue Service (SQS) to CloudTrail SNS topics](#subscribe-simple-queue-service-sqs-to-cloudtrail-sns-topics)
  - [Setup SplunkAppForAWS](#setup-splunkappforaws)
  - [Billing and usage module](#billing-and-usage-module)
  - [CloudTrail module](#cloudtrail-module)
  - [Troubleshooting](#troubleshooting)
- [Reference: what is CloudTrail](#reference-what-is-cloudtrail)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

This document describe how to use a script to setup AWS Cloudtail services to audit API calls, and how to setup SpunkAppForAWS app to generate reports in Splunk.

## Prerequisites for this setup

All the setups can be done through AWS console, but we use a script for CloudTrail setup to ensure naming schema consistency cross
all regions. All AWS services (Cloudtrail, SNS, SQS) created will have a prefix __'accountId-'__

### Install AWSCLI and Jq

  [AWSCLI] (https://github.com/aws/aws-cli) command line tool is used to create Cloudtrail. To install (or upgrade) the package:

```
$ brew install awscli jq
```

   This will install _aws_ command under /usr/local/bin. There are three ways to setup AWS CLI AWS credentials. The examples here assumes you run the Cloudtrail creation code on an on-premise system and use a configuration file for key id and key secret. If you run it on EC2, you need to create an IAM role and åthe aws cli can use role-based token automatically.
    
   To configure AWSCLI with an IAM user that can create CloudTrail, SNS, SQS, and S3 bucket services
    
```
$ aws --profile <profile> configure
```

   The above command will generate two files, for example:
   
```
$ cat /etc/.aws/awscli.conf
# For AWS CLI
[profile mylab-cloudtrail]
region = us-west-2
...

$cat /etc/.aws/credentials
[mylab-cloudtrail]
aws_access_key_id = <key id>
aws_secret_access_key = <secrect>
```


## Quick start: Create AWS CloudTrail

For security purpose, we monitor all regions that CloudTrail service is available. These are the [current supported regions](http://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-supported-regions.html):

* us-east-1
* us-west-1
* us-west-2
* eu-west-1
* sa-east-1
* ap-northeast-1
* ap-southeast-1	
* ap-southeast-2

To enable CloudTrail on these regions, download and run [cloudtrail-admnin.sh](./scripts/cloudtrail-admin.sh). This script calls awscli cloudtrail command to create or destroy cloudtrail setup. For setup, it performs the following functinons:

* Turn on Cloudtrail service in all supported region to send reports to *s3://`<accountId`>-cloudtrail* S3 bucket
* Create trail with name *s3://`<accountId>`-cloudtrail-`<region>`*
* Create Simple Notification Service (SNS) topic for the Cloudtrail service in each region
* (Optional) Create *`<accountId>`-cloudtrail* SQS in one region to receive CloudTrail report notification. You only need this for Splunk.
  
All CloudTrails reports are sent into one S3 bucket default to *s3://`<accountId`>-cloudtrail*. Global events - generated by global services such as Identity and Access Management (IAM) - will be logged in the trail in the region given at the command line.

For help:

```
$ ./cloudtrail-admin.sh -h
create-cloudtrail -a <action> -p <profile> [-b <bucket>] -r region [-n] [-y]

 -a <create|show|delete>: action. create, show or delete cloudtrails setup by this tool.
 -p <aws profile>: authenticate as this profile.
 -b <bucket>: optional. bucket name to get all trail reports.
 -r <region>: region to get AWS global events, e.g. IAM
 -y     : non-interative mode. Answer to yes to all default values.
 -n     : dryrun. print out the commands
 -h     : Help
```

To create Cloudtrails, SNS and SQS:

```
./cloudtrail-admin.sh -a create -p <aws profile> -r <region to receive global event>
```

To show Cloudtrails, SNS and SQS:

```
./cloudtrail-admin.sh -a show -p <aws profile>
```

To tear down the resources created:
```
./cloudtrail-admin.sh -a delete -p  <aws profile> -r <region to receive global event>
```

NOTE: The Cloudtrail bucket will not be deleted since it may contain past audit logs. 

The cloudtrail now is enabled and you can go to AWS CloudTrail service console to view logs or through API calls. 


## Third party visualized reporting tools
Many tools are available to generate visualized reports using the CloudTrail files stored in S3 bucket. Here are listed 
[AWS partners](http://aws.amazon.com/cloudtrail/partners/). This documentation describes how to use [SplunkAppforAWS](http://apps.splunk.com/app/1274/) to consume Cloudtrail data and generate reports. 

## The CloudTrail and Spunk integration

### Why use CloudTrail
Traditionally system administrators monitor a system's integrity using intrusion detection tools such as Tripwire. System logs are usually sent to 
a central log server for auditing and security analysis.

For services running on AWS, another important operation and security related auditing is to monitor API calls that can change services and environments. Use cases enabled by CloudTrail service:

* Security Analysis
* Track Changes to AWS Resources
* Troubleshoot Operational Issues
* Compliance Aid

Here is an example of API call recorded by CloudTrail and how Spunk reports it (what, when, who, where, etc.)

	{ [-]
	   awsRegion: us-west-2
	   eventID: 7ad60379-0e2e-4b6c-8a6f-100f00fc5df1
	   eventName: ModifyReservedInstances
	   eventSource: ec2.amazonaws.com
	   eventTime: 2014-07-14T19:29:42Z
	   eventVersion: 1.01
	   requestID: 900f0797-0f2d-43dd-bcf5-41e1cfbc5c9f
	   requestParameters: { [-]
	     clientToken: fa8494bb-8546-4fc4-9bb8-8dc84e7d0015
	     reservedInstancesSet: { [+]
	     }
	     targetConfigurationSet: { [+]
	     }
	   }
	   responseElements: { [-]
	     reservedInstancesModificationId: rimod-f0713af8-0984-42ff-8016-3dfea813b209
	   }
	   sourceIPAddress: 171.66.221.130
	   userAgent: console.ec2.amazonaws.com
	   userIdentity: { [-]
	     accessKeyId: XXXXXXXXXXXXXXXXXXX
	     accountId: XXXXXXXXXXXXX
	     arn: arn:aws:sts::XXXXXXXXXXX:assumed-role/admin-saml/john
	     principalId: XXXXXXXXXXXXX:jonh
	     sessionContext: { [+]
	     }
	     type: AssumedRole
	   }
	}

### Create an IAM user for Splunk integration

   You should create an IAM user with the minimum access privileges needed. 

   Fexample, you can create an IAM user _cloudtrail-splunkapp_  which has permission to read SQS, delete message from SQS, and get data from S3 bucket. The following polices should work if you use canned AWS policy generator:

* Readonly access to S3 bucket
* Readonly access to CloudTrail
* Full access to the SQS (it deletes messages after read stuff from the message queue)

In this integration, we create CloudTrail service for each region and Simple Notification Service (SNS) topic for each CloudTrail service. The reports from all regions are aggregated to one S3 bucket. One Simple Queue Service (SQS) is subscribed to all the SNS topics. 

![](./images/splunk-aws-integration.png)


### Subscribe Simple Queue Service (SQS) to CloudTrail SNS topics

The SQS named *`<accountNumber>`-cloudtrail* is created by cloudtrail-admin.sh script. It is easiest to use AWS's SQS service console to subscribe the queue to each SNS cloudtrail SNS topics because AWS service will generate proper policy for the queue to allow messages from the SNS topics.

SQS is needed in Splunk AWS app configuration. Splunk AWS app runs at defined interval to retrieve messages from AWS SQS service. The message body contains the S3 bucket location for Cloudtrail reports. Splunk then calls S3 API to get Cloudtrail reports from the S3 bucket.

### Setup SplunkAppForAWS

The application has two major report categories - Cloudtrail monitoring for API activities, and Billing and Usage that helps budgeting and cost analysis. 

### Billing and usage module

This module requires to turn on [Detailed Billing Reports](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/detailed-billing-reports.html). AWS will publish detailed billing reports in CSV format to the S3 bucket that you specify. If you have consolidated billing setup, ask the payer to create an IAM user that has read access to the S3 bucket that collects billing csv files. If you want to monitor the subaccounts's running status (e.g. the break downs of running on-demand instances, reserved instances, type of instances etc.), you need to create an IAM user in each sub-account that has access to check EC2 status in that subaccount. 

To make the Billing and Usage work, you need the following:

1. If you use consolidated billing, ask the payer to create an IAM user that have read only access to the S3 bucket's billing CSV file
2. If you need to pull dynamic EC2 information, create an IAM user for each account/subaccount with the following policies:
*  Amazon EC2 Reports Access
*  Amazon EC2 Read Only Access
3. opt/splunk/etc/apps/SplunkAppforAWS/lookupsinstancetype.csv
   This file should contain all instances types that you use in your account. The list from the SplinkAppforAWS may not be complete.
4.  aws.conf file 

  The file contains the setup for Billing and Usage. Here an example:

        # more /opt/splunk/etc/apps/SplunkAppforAWS/local/aws.conf

        # All this three stanzas are required, in order to run AWS App.

		[keys]

		# Format :
		# <accountno> = <id> <access-key> <secret> <limit>
		#
		# Where:
		#   id = company/group name without space
		#   access-key = AWS Access Key
		#   secret     = AWS Secret Key 
		#   limit      = Monthly spending limit for this account

        # If you have this
		012345678901 = <rollup-paying account> <IAM billing policy user key> <IAM billing policy key secrete> 200000

        # Accounts/subaccounts. Information can be found in the detailed billing reports
		<account-no1> = <account-name1> <IAM EC2 policy user key> <IAM EC2 policy key secrect> 100000 
		<account-no2> = <account-name2> '' '' 100000

		[regions]

		rgn1 = eu-west-1
		rgn2 = sa-east-1
		rgn3 = us-east-1
		rgn4 = ap-northeast-1
		rgn5 = us-west-2
		rgn6 = us-west-1
		rgn7 = ap-southeast-1
		rgn8 = ap-southeast-2

		[misc]

		# Format :
		# corpkey = <id> <access-key> <secret>
		#
		# Where:
		#   id = company name/corp account name without space
		#   access-key = AWS Access Key
		#   secret     = AWS Secret Key

        # Same as you specified in [key] section
		corpkey = <rollup-paying account name> <IAM billoing user key> <IAM billing user key secret>

		# acno is corp account number for AWS, same as in [key] section
		acno = 01234567891

		# s3bucket is bucket name where AWS bill csv files will be dropped
		s3bucket =  <builling bucket>  
		
5 .  /opt/splunk/etc/apps/SplunkAppforAWS/lookups

   This file contains account and name mapping. Put accounts/subaccounts found in the billing CSV file here:

            # more  accountNames.csv
			accountId,accountName
			012345678901,<name>
			123456789012,<name>
			.....

### CloudTrail module

The SplunkAppForAWS needs to be installed on Splunk search head and indexers. The data input only needs to be configured on indexers where the app will retrieve audit reports and build the indexes. Splunk search head does search against the logindex servers. 

You can setup data input on Splunk Settings->Data Input console:

* Go to "Settings->Data Inputs" 
* Create a new data input
* Fill out the CloudTrail IAM user's key, secret, SQS name, and the region, created by create-cloudtrail.sh script.
* Set script interval to 300 seconds. Cloudtrail delivers report within 15 minutes of API calls to S3 bucket, and within 5 minutes to SNS after the report is posted. You have up to 20 minutes to see the the latest API calls to show up in Splunk.
* In the "Script iterations" box, use a very large number, otherwise, the app script will stop running after 2048 runs.
* Click "More Advanced Setting", check Source Type to Manual, and "Source" to aws-cloudtrail

It will generate a __inputs.conf__ file, that can be managed by a configuration tool such as Puppet. Here is an input.conf file example:

    splunkidexer:/opt/splunk/etc/apps/launcher/local# more inputs.conf 
    exclude_describe_events = 0
	index = aws-cloudtrail
	interval = 30
	remove_files_when_done = 0
	key_id = <access key>
	secret_key = <access key secret>
	sqs_queue = <accountNumber>-cloudtrail
	sqs_queue_region = us-west-2
	sourcetype = aws-cloudtrail
	script_iterations = 1000000
	
### Troubleshooting

* First update to the latest boto library. The package comes with it boto installation under /opt/splunk/etc/apps/SplunkAppforAWS/bin directory. To update:

        mkdir /tmp/boto
        pip install --install-option="--prefix=/tmp/boto"  boto
        cd /tmp/boto/lib/python2.7/site-packages
        tar cvf /tmp/botolib.tar boto

        cd /opt/splunk/etc/apps/SplunkAppforAWS/bin
        mv boto boto.old && tar xvf /tmp/botolib.tar  
        
* /var/log/splunk/splunkd.log file contain splunk logs. Grep 'AWS' keywords to find out if the scripts are running and if there are any problems.

* $SPLUNK_HOME/etc/apps/SplunkAppforAWS/lookups directory should contain billing data pulled from S3 bucket. subaccounts.* files are generated from the get_bill.py program each time a pulling occurs. 

* $SPLUNK_HOME/etc/apps/SplunkAppforAWS/log contains logs generated by various scripts that pulls billing, instance usage information. You need access key and key secret (create an IAM user for it) for the accounts you want to pull the information. 

## Reference: what is CloudTrail
[AWS CloudTrail] (http://aws.amazon.com/cloudtrail/) is an AWS service. When enabled, it captures AWS API calls made by or on behalf of an AWS account and delivers log files to an Amazon S3 bucket. 
 
