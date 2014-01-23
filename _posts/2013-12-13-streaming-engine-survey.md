---
layout: post
title:  "Publishing MongoDB Metrics to CloudWatch in Amazon AWS"
date:   2014-01-14 18:00:04
categories: cloudwatch amazon aws mongodb devops autoscaling devops
---

##Summary
Amazon [CloudWatch](http://aws.amazon.com/cloudwatch/) is a useful tool for monitoring your compute resources, services, and applications running in AWS. Amazon automatically publishes metrics for you when you use many different AWS Services. 

In this Article I will show how to publish custom metrics to CloudWatch for monitoring MongoDB. These metrics can be used to drive policy for [AutoScale Groups](http://aws.amazon.com/documentation/autoscaling/) to automatically scale up and down your compute resources based on your custom metrics.  

##Setup
In this example I am assuming use of the [Amazon Linux AMI](http://aws.amazon.com/amazon-linux-ami/). Steps should be similar on any Linux AMI.

For the scripting language to compute the metrics, we will use Python. Python works well for several reasons:

1. It's easy to call AWS command line tools
2. It's easy to script with minimal OS dependencies
3. Libraries/Drivers exist to calling services like MongoDB

###IAM Role
You need to grant your EC2 system rights to publish data to CloudWatch. An easy way to do this is via an IAM Role.

Below is an example of a Policy that allows your EC2 instance to publish metrics to CloudWatch

	{
  		"Version": "2012-10-17",
  		"Statement": [
   		 {
   		   "Sid": "Stmt1389897768000",
    	   "Effect": "Allow",
    	   "Action": [
       			 "cloudwatch:GetMetricStatistics",
       			 "cloudwatch:ListMetrics",
     		     "cloudwatch:PutMetricData",
       			 "ec2:DescribeTags"
     	    ],
      		"Resource":  "*"
      
        }
 	    ]
	}
	
Simply either modify your machines existing IAM Role or create a new role and add this policy. The IAM Role must be associated with the ec2 instance at the time of creation.

###EC2 Install Pre-requisites
Python is already installed on the Amazon Linux AMI. All we need is the [PyMongo](http://api.mongodb.org/python/current/) driver. The easiest way to get this without other dependencies is to use easy_install

	easy_install pymongo
This can be added to a bootstrap script in your UserData section of [CloudFormation](http://aws.amazon.com/cloudformation/)
	
###Write the Metric Script
Now we just need to write a simple script that will perform the mongo query, extract the metric, and then publish the metric to CloudWatch. 


	#Import the required libraries for the script to run
	import commands
	import json
	import pymongo
	from pymongo import MongoClient

    #Get the instanceId for our machine. This is important later for
    #autoscaling. The dimensions we select here when publishing
    #must be matched later by our autoscale policy
	ret, instanceId = commands.getstatusoutput("wget -q -O - http://169.254.169.254/latest/meta-data/instance-id")

	#Connect to MongoDB
	#Here we run this script where Mongo lives so we connect to localhost
	client = MongoClient('localhost', 27017)

	#Choose the database where your data lives, here a test DB
	db = client.test
	#Select the collection. Here we select the jobs collection
	collection = db.jobs

	#Perform the query you want as the basis of the metric
	#Massage the data all you want in Python before reporting
	count = collection.find({"status": "New"}).count()

	#Publish the Metric to CloudWatch	
	cmd = "aws cloudwatch put-metric-data --metric-name JobCount --namespace ExampleOrg --dimensions \"instance=" + instanceId + ",servertype=MongoDB\" --value " +  str(count) + " --region us-east-1"

	ret,cmdout= commands.getstatusoutput(cmd)

	##Cleanup
	client.close()


Park this somewhere on your image where it won't be deleted. For example you can drop it in a folder like /opt/metrics/myMetric.py 

Consult the [CloudWatch Documentation](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/cloudwatch_concepts.html) for the different options you can provide for your metric. You can specify additional paramters such as the Unit of Measure, timestamp, and dimensions. Note that dimensions are very important later for autoscale policies as the dimensions of your metric and the policy must match. 

##Collecting the Metric
Once we have our mertric, we simply need the system to call it. We can accomplish this easily via a cron job.

For example, to do this create a file for your job in /etc/cron.d such as /etc/cron.d/mongoMetric

Inside include the following:

		* * * * *	root	/usr/bin/python /opt/metrics/myMetric.py


##Viewing the Metric
Your metric will show up in the Amazon CloudWatch service console under custom metrics organized by your namespace. The dimensions you specified in the put-metric-data script will also be available as part of the metric table.

##What's Next?
From CloudFormation or the CloudWatch Console, you can create an Alarm where the action is either a notification or an Autoscaling Policy action. The policies for autoscaling are separate from the Cloud Watch alarms so you can have multiple metrics or alarms all call the same policy.