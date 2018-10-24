# Google Cloud - Compute Engine - Lab

This lab is part of the [foundations training](https://github.com/ealliaume/foundations) at [OCTO Technology Australia](http://careers.octo.com.au/).

## Overview of this lab

<img src="static/overview.png" />

In this quick lab, we will play with the Google Cloud Compute Engine.

At the end of the lab you will be able to:
- **create buckets and files** on Google Cloud Storage, and make some files public
- **create a compute engine instance** on Google Compute Engine
- **create instance templates** and **instance groups** and play with auto scaling
- configure and use a **multi-regions load balancer**

This lab should take approximately 60 minutes.
You will need to use your own google cloud account.

## What is the Google Compute Engine

Google Compute Engine (GCE) is the Infrastructure as a Service (IaaS) component of Google Cloud Platform. 
Google Compute Engine enables users to launch virtual machines (VMs) on demand. 
VMs can be launched from the standard images provided or by using custom images created by users. 

Let's start with a bit of terminology, your applications will use:
- vm, a virtual machine
- instance templates, convenient way to save a VM instance's configuration so you can use it later to create new VM instances or groups of VM instances
- managed instance groups, which uses an instance template to create a group of identical instances. Compute Engine offers two different types of instance groups: managed and unmanaged instance groups, we will focus on managed instances group here. 

## Target Architecture

<img src="static/target-architecture.png" />

## The Lab - step by step

### 0. Prerequisites

#### Create a project on GCP and activate billing

- Google to the [Google Cloud console](https://console.cloud.google.com)
- Click on Project List (in the header next to the Google Cloud Platform Logo), the "Create Project"
- Give it a "Name", select a billing account, then use the "Create" button

For more information about this prerequisite step please refer to the [official documentation](https://cloud.google.com/resource-manager/docs/creating-managing-projects#creating_a_project).

### 1. Create a bucket

#### Upload bootstrap files in simple storage and expose it in HTTP

* Create a bucket (Gcp menu > Storage > Storage)

* Upload the following 2 files which you can find in the _labs-files_ directory:
  * Belgium configuration: _frontend-belgium.py_
  * Sydney configuration: _frontend-sydney.py_

  We could use exactly the same file in different region, the only reason we are 2 of those it to be able to customize the messages served by the python endpoints. 

* Make objects public using the cloud shell. To do this click the 'activate cloud shell' terminal in the toolbar and run the following commands:
  * `gsutil acl ch -u AllUsers:R gs://era-boot/frontend-belgium.py`
  * `gsutil acl ch -u AllUsers:R gs://era-boot/frontend-sydney.py`

The syntax for accessing resources is:
`gs://[BUCKET_NAME]/[OBJECT_NAME]`. For more information on the cloud shell and gsutil see : https://cloud.google.com/storage/docs/gsutil


### 2. Set up instance groups and templates

#### Create instance template for Sydney

Instance templates (Compute engine > Instance Templates):

We will launch our instance from an Instance Template, we will need two of these because there are two bootstrap actions (one to set up the sydney instance, and one for belgium)

* 1 CPU (default)
* Debian (default)
* Firewall > Allow HTTP Traffic
* Startup script (Automation > Startup script): `wget https://storage.googleapis.com/era-boot/frontend-sydney.py
sudo python frontend-sydney.py &`
* Leave all other options as default

#### Create the second instance template for Belgium using “copy”

Click on the template and then use the 'copy' option from the toolbar, this will load the previous selections. Then just change the startup script as below.

`wget https://storage.googleapis.com/era-boot/frontend-belgium.py
 sudo python frontend-belgium.py &`

#### Create instance Group for Sydney

Instance Groups (Compute Engine > Instance Groups)

Create an instance group with the following settings (leave as default those not mentioned)

* Single Zone
* Region: Sydney
* Select the correct instance template
* Autoscaling: ON
* Autoscaling: CPU usage!
* Healthchecks: create a new healthcheck, Protocol:  HTTP 80, Health critieria: 5 5 2 2
* Initial Delay: 120s

#### Create Client Instance Syd

Click on instance template > create VM to create a new VM from the template
The point of creating client instances is to be able to ssh in to these clients and call the backend service from each of these two regions.

* Name: era-client-syd
* Machine type: Micro
* Region: Sydney
* Allow HTTP traffic


#### Create Client Instance Belgium

* Name: era-client-bel
* Machine type: Micro
* Region: Belgium
* Allow HTTP traffic

#### SSH on both Client instance using interface

(VM Instances > Connect > SSH)

* Curl internal IP of instance group
  * curl http://10.152.0.2/
  
You should see 'Hello from Sydney' and 'Bonjour from Europe' from the different instances

### 3. Set up Load Balancers

#### Create HTTP Load Balancer

(Network services > Load Balancer)

* Create a http load balancer and assign a name.

* Create a backend service and assign the details of your instance group.
  * Add healthcheck group
  * Add instance group
  * Set CPU utilisation to 60%

* Don't add anything for Host and Path Rules

* Create Frontend configuration
  * Era-front-1
  * Http
  * Ip v4
  
Review, finalise and create. This will take about 5 minutes!

Check the load balancer has been created (swap with your IP):
`curl http://35.241.43.206/`

#### Check Monitoring Group > Monitoring Tab

### 4. Autoscale!

#### Play with Autoscaling

Do this with the following command (change IP address):

* Then raise the CPU from local shell
* `while true; do curl http://35.241.43.206/service ; done`
* Check monitoring  => menu instances
* Then kill bash loop
* Wait for the cluster to calm down

#### Create instance group for Belgium

* Simple Zone
* Region: Belgium
* Select good instance template
* Autoscaling: ON
* Autoscaling: CPU usage!
* Healthchecks:   era-hc    HTTP 80   /   5 5 2 2
* Initial Delay: 120s

#### Add the new Instance Group to the Load Balancer

`gcloud compute backend-services add-backend era-backend-bel-42 --instance-group=era-ig-bel --region=europe-west1`

* You now have 1 Load balancer service with 2 backends!

## Success!!!

Be sure to delete all resources used!

<img src="./static/success.gif" width="500" />
