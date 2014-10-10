---
layout: post
title:  "Continous Delivery with Docker on AWS"
date:   2014-10-10 18:30:04
categories: docker amazon AWS devops continuous delivery
---

##Summary
Over the past few years, the dream of **Continuous Delivery** (CD) is easier to achieve due to the rapid advancement of DevOps automation tools. I've achieved various levels of continuous delivery in the past in a variety of environments using tools such as Chef but there has always been some brittleness and the process required maintenance and never felt ideal.

Recently, at OpenWhere I was able to develop a continuous delivery process utilizing Docker which feels much closer to realizing a Continuous Delivery *utopia*. 

The process leverages several components mainly Docker, Jenkins (for Orchestration), CloudFormation (for Infrastructure Automation), and bash. This blog will cover the overall Continuous Deployment Architecture and overall approach and assumes some basic familiarization with [Docker](https://www.docker.com/) and Jenkins. 

Overall, Docker what Docker brings to this process which is new is that you can now put your code and all its requirements in a **black box** which is deployable across environments and even operating systems. How you mange the CD of the boxes doesn't change much despite whatever is inside and its complexity. 

In contrast, with a provisioning approach using tools like Chef, you end up needing to adjust your process for each technology you are deploying. 

- **Examples**: 
	- How do Python deployments work? 
	- How do I hot swap a Java war, etc?

##Setup
The chart below depicts the components of our *Continuous Delivery* Infrastructure.

![Streaming Survey (continued)]({{ site.url }}/assets/Continous_Deployment_Docker.jpeg)

Jenkins is the heart managing the Process. We actually structure the Jenkins build into 3 jobs per project.

- **Code Build Job** *(Continuous Integration)* for the code
- *On Success*, **Docker Container Build Job** which is your new deployment artifact and pushes it to a Docker Registry
- *On Success*, **Docker Deployment Job** which triggers your servers that have this deployment artifact to update

The benefit of making the jobs independent is that you can re-deploy separately from building the code, etc.

The actual build jobs steps we capture and version control in bash scripts. The bash scripts live with the actual code so each project knows how to build itself. This makes each Jenkins job relatively simple and uniform so there is no secret sauce out of version control in Jenkins.

A Docker Registry is used to host the deployment artifacts. A Docker image contains the built code artifact along with the version controlled settings needed to run. Also, its normally super easy to set up a new product, since Docker has the concept of layers (other images). All Java war applications can share a base Tomcat layer, all Python projects a Python base layer etc.

For the actual deployment, a bash script initiates password-less ssh to the EC2 boxes found via AWS tags. We use the tag "DeployGroup" which is tied to a unique Jenkins job associated with a Git Repository. An alternative would be to use an IAM role and simply kill and build a new box, which would then come up with the new Docker Image.

CloudFormation covers the actual infrastructure deployment. The cloud init section of the EC2 box installs Docker and pulls down the appropriate Docker image and launches it as a "container" with the appropriate configuration. This was also much simpler than our previous Chef approach which required setting up certificates, assigning a role, then bootstrapping the box. 


##Conclusion
Docker has been gaining traction and maturity at a rapid rate. I am expecting new features to make using Docker and this process even easier. There was definitely some initial growing pains in setting up this process but the end result has paid many dividends due to the benefits of continuous delivery which include things like:

- Faster deployments
- Less headaches for DevOps team (or individual)
- More frequent integration
- Save time and headaches!