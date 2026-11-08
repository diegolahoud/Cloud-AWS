### Secure & Scalable Three-Tier Application in AWS

## Overview
The objective of this project is to create a secure and scalable three-tier web application in AWS. The infrastructure is deployed in the **us-east-1** region within a custom Virtual Private Cloud (VPC) using the `192.168.0.0/24` CIDR block. To ensure high availability, the VPC spans two Availability Zones (**us-east-1a** and **us-east-1b**), each containing four distinct subnets. To ensure the Least Privilege principle, this lab will have 4 different users, with 4 different level of access so each users will be added to a Group where they will be only have the exact amount of access they need.

## VPC and Subnets

**1) us-east-1a**
* **AZA-WebTier-Public:** `192.168.0.0/28`
* **AZA-AppTier-Private:** `192.168.0.16/28`
* **AZA-DBTier-Private:** `192.168.0.32/28`
* **AZA-Reserved-Private:** `192.168.0.48/28`

**2) us-east-1b**
* **AZB-WebTier-Public:** `192.168.0.64/28`
* **AZB-AppTier-Private:** `192.168.0.96/28`
* **AZB-DBTier-Private:** `192.168.0.112/28`
* **AZB-Reserved-Private:** `192.168.0.128/28`

## IAM Users & Groups:
**Note:** This lab used standard IAM Users, and not the new AWS Identity Center. Using AWS Identity Center is the new best practice of AWS but for this case it was not used. 

* **User** Dev1 | **Group:** DevGroup | **Group Access Level:** AmazonEC2ReadOnlyAccess, AmazonS3FilesClientReadWriteAccess, and CloudWatchEventsReadOnlyAccess. 

* **User** Ops1 | **Group:** OpsGroup | **Group Access Level:** AmazonEC2FullAccess, AutoScalingFullAccess, ElasticLoadBalancingFullAccess, and AmazonRDSReadOnlyAccess

* **User** admin-projects (main account) | **Group:** AdminGroup | **Group Access Level:** AdministratorAccess

* **User** Auditor1 | **Group:** AuditorGroup | **Group Access Level:**  SecurityAudit, and ViewOnlyAccess

## AWS Security Groups:

**1. alb-sg (Public Load Balancer)**
**2. ec2-sg (App Tier)**
**3. db-sg (Database)**
**4. vpce-sg (VPC Endpoints)** 

## AWS Services Used

- **Amazon Route 53 (R53)**
- **Amazon CloudFront CDN + Web Application Firewall (WAF)**
- **Internet Gateway (IGW)**
- **Public Application Load Balancer (ALB)**
- **Amazon EC2 (Web Tier):** 2 instances (Names and specs TBD)
- **Internal Application Load Balancer (ALB)**
- **Amazon EC2 (App Tier):** 2 instances (Names and specs TBD)
- **Amazon RDS Database:** 1 Primary Database in AZ1
- **Amazon RDS Database:** 1 Standby using "Multi-AZ"
- **VPC Endpoints:** S3 Gateway Endpoint & Interface Endpoints
- **Auto Scaling Group (ASG)**

## Infrastructure Creation

### 1) VPC, Subnets, and Internet Gateway (IGW) Creation

**VPC Setup**
1. Go to the AWS services search bar and type **VPC** to navigate to the main VPC console.
2. In the upper right corner, click on **Create VPC**.
3. Select **VPC only**.
4. Fill out the **Name tag** (for this lab, use `3-Tier-App-Project`).
5. Input the IPv4 CIDR previously selected for the VPC (`192.168.0.0/24`).
6. Click **Create VPC**.

**Subnet Setup**
1. After creating the VPC, click on **Subnets** in the left-hand navigation pane and click **Create subnet**.
2. Select the `3-Tier-App-Project` VPC from the dropdown menu.
3. Add the **Subnet name** (e.g., `AZA-WebTier-Public` for the first one).
4. Select the **Availability Zone** (e.g., `us-east-1a`).
5. The IPv4 VPC CIDR block should automatically populate. Enter the specific subnet range in the **IPv4 subnet CIDR block** field (for the first one, use `192.168.0.0/28`).
6. Click **Add new subnet** and fill out the respective information for the remaining subnets.
7. Once all subnets are configured, click **Create subnet**.

**Create and Attach the Internet Gateway (IGW)**

> **Note:** This process will give the VPC access to the public internet. 

1. On the VPC dashboard, click **Internet Gateways** on the left. 
2. Click **Create internet gateway** and set the name as `3-TierApp-IGW`.
3. Click **Create internet gateway**.
4. After creation, click on **Actions** in the top right, then click on **Attach to VPC** and select `3-Tier-App-Project`.

**Configure the Route Tables**
> **Note:** This will make the web tier subnets public. You must set the target to the IGW because Route Tables operate at Layer 3 (Network) to route packets, while your ALB operates at Layer 7 (Application).

1. On the left side of the VPC menu, click on **Route Tables**, then click **Create route table**.
2. Set the name as `Public-RT-3TierApp`, select the VPC `3-Tier-App-Project`, and click **Create route table**.
3. At the bottom of `Public-RT-3TierApp`, click on the **Routes** tab, then **Edit routes**.
4. Click **Add route**, enter `0.0.0.0/0` as the destination IP, select **Internet Gateway** as the target, choose `3-TierApp-IGW`, and click **Save changes**.
5. Click on the **Subnet associations** tab, then **Edit subnet associations**. Select the two Web-Tier subnets across both AZs and save.
6. Follow the steps above to create a second route table named `Private-RT-3TierApp` for the remaining private subnets. **Do NOT add an IGW route**, as these subnets must remain private. Associate all App, DB, and Reserved subnets to this table and save.


### 2) Create Security Groups

> **Note:** This implements Security Group Chaining to ensure no tier can be bypassed, keeping backend servers completely isolated from the internet.

Navigate to **Security Groups** (under the VPC console, on the left side) and create the following four groups inside your `3-Tier-App-Project` VPC:

**1. alb-sg (Public Load Balancer)**
- **Description:** This is the ALB that receives the Internet traffic from the IGW to the public tier
- **VPC:** `3-Tier-App-Project`
- **Inbound Rules:** HTTP (80) and HTTPS (443) from `My IP`. (This will for the lab)
- **Outbound Rules:** All traffic (default).

**2. ec2-sg (App Tier)**
- **Description:** This SG is for the app tier so it only accepts traffic from the alb-sg **TECNICAL ERROR: THIS IS THE SG FOR THE APP-TIER, BECAUSE I FOUND OUT LATER IN THE PROJECT I CANNOT DELETE THE SG SO THE DESCRIPTION IS WRONG BUT THE INBOUD RULE IS CORRECT*
- **VPC:** `3-Tier-App-Project`
- **Inbound Rules:** Create one inbound rules for HTTP, port 8080, source type "Custom" and the source as the `internal-alb-sg`
- **Outbound Rules:** All traffic (default).

**3. db-sg (Database)**
- **Description:** This security group will be for the database tier to only accept traffic from the ec2-sg
- **VPC:** `3-Tier-App-Project`
- **Inbound Rules:** Type MySQL/Aurora on port 3306, and the source `ec2-sg`
- **Outbound Rules:** All traffic (default).

**4. vpce-sg (VPC Endpoints)**
- **Description:** Creates a security group from the vpc endpoints and ec2
- **VPC:** `3-Tier-App-Project`
- **Inbound Rules:** Type HTTPS on port 443, and source `ec2-sg`
- **Outbound Rules:** All traffic (default).

**5. internal-alb-sg**
- **Description:** Internal ALB for the app tier. It allows HTTP traffic from web tier group
- **VPC:**  `3-Tier-App-Project`
- **Inbound Rules:** Allow HTTP traffic on port 80 from the source web-tier-sg
- **Outbound Rules:** All traffic (default)

**6. web-tier-sg**
- **Description:** sg for the web tier. receives traffic from the public ALB
- **VPC:**  `3-Tier-App-Project`
- **Inbound Rules:** Allow HTTP and HTTPS port 80 and 443 from the custom source: alb-sg
- **Outbound Rules:** All traffic (default)

### 3) Create Secret Managers

> **Note:** The Secret Manager works as a Password Manager but instead of using it for personal cases, it is used for AWS infrastructure services so developers do not leave a hard-code credentials in the source code. In this case it is more secured. 

1. Open "Secrets Manager" in the AWS dashboard.
2. Click on "Store a new secret"
3. Select "Other type of secret" 
4. Under the Key/value pairs you need to choose the credentials for the username, and password.
    key:username | value: <username>
    key:password | value:<password>
5. Leave "Encryption key" as default, and click next
6. Select the name of the secret, in this case it is "3tierapp-db-credentials"
7. Add description: Master credentials for the upcoming RDS Database
8. Leave the Automatic Rotation as disabled. 
9. Click next, and then store. 

### 4) Create the Database

> **Note:** There are two types of Relational Databases in AWS, each with their own benefits and downsides: 
>
> **A) Amazon Aurora:** It is built specifically for the cloud, offers high performance, and is highly available by default (it automatically replicates data 6 times across three AZs). There is only one primary writer but can be many read replicas. If the primary fails, a standby automatically takes over. The downside is that it is more expensive than standard RDS.
>
> **B) Amazon RDS:** It operates like a traditional database and supports engines like MySQL, PostgreSQL, MariaDB, and SQL Server. By default, it is just a single virtual server with a single hard drive attached to it. It does not replicate by default. To make it highly available, you must either enable "Multi-AZ" (which creates a standby server in another availability zone) or manually create "Read Replicas" (which provisions a new server with a separate hard drive).
>
> You must analyze your application's requirements to determine which database is the best fit. For this project, we selected **Amazon RDS** because we need to provision a database that is secure but highly cost-effective. Amazon RDS is free tier eligible but because we are choosing "Multi-AZ", AWS needs to provision another instance in another availability zone which incurs charges. 

! Create the Subnet Group

1. Go to "Aurora and RDS" in the AWS services.
2. On the left menu click "Subnet Groups". 
! We click on the "Subnet Group" so we can select the subnets in where we are going to create the databases instances. They are good for network isolation, and for Multi-AZ requirement as we need to select the two AZ in where the primary RDS and standby where be located.
3. Select the name as "subnet-group-3tierapp", the description as "Subnet group for the database tier, and the VPC as "3-Tier-App-Project". Note: The name cannot start with a number
4. Select the Availability Zones you need the two RDS instances, in this case for the project we are selection "us-east-1a" and "us-east-1b"
5. Select the subnets dedicated for the databases instances. AZA-DBTier-Private, and AZB-DBTier-Private

! Create the Database Instances

1. On the Aurora and RDS menu, click on the left side on "databases". Create Database. And click on "Full Configuration"
2. On the Engine option select "MySQL".
3. For templates choose "Dev/Test. This is a learning project lab so we do not need production graded environment to keep lower cost, and still high availability.
4. For Availability and Durability choose "Multi-AZ DB instance deployment"
5. Leave the engine version as the default "MySQL 8.4.11". The DB instance idenfifier add "three-tier-db"
6. Type the master username you selected in the Secrets Manager in the step before. Leave the encryption type as the default. 
7. For the Instance Configuration, click on "Bustable classes", and select the "db.t3.micro".
! We selected a Burstable Class because this is the most economic option for our project while still keeping performance. The Burstable Class will have a baseline when it is not being used much, and once more queries are showing up, it will increase the capacity and handle all the requests so it is a good option for a budget friendly enviroment. The standard classes, they are at high performance all the time despite not being in used so it will cost more even if the database is not being used. 
8. Leave the storage as "General Purpose SSD gp3" as default, change the Allocated Storage to the minimum (20 GiB), and go to Additional Storage Configuration, and uncheck "Enable Storage Autoscaling" to prevent any unexpected charge due to storage increase. We also selected the 20 GiB to reduce costs.
9. For connectivity, select the "3-Tier-App-Project" VPC. Select the "subnet-group-3tierapp" as the DB subnet group. Select "No" public access. And for the VPC security group (firewall), select the "db-sg(database") and make sure to remove the default. 

**Note regarding databases, and costs:** AWS bills by the hour, for the purpose of this project we will be turning off the database periodically but it is not recommended in production environment. We are allow to turn off the database a period of 7 days before turning it back on again.   

### 5) Create the Auto Scaling Groups (ASG)

> **Note:** Auto Scaling Groups are the ideal solution to make your application scalable. If the applicaiton goes viral, it will scale the EC2 instances on the Web-Tier, and the App-Tier. Because the project will have two ASG, if the web-tier becomes busy but the app-tier is not busy, it will only scale the web-tier. The objective of the ASG is to scale the application horizontally, that means, adding or removing EC2 instances without changing the features. 

## Part A) Create IAM role so the App Tier can communicate with the databases.

1) Go to the IAM menu in AWS, select Role, and click on "Create Role"
2) Click on AWS Service, the Service select EC2, and click Next. 
3) Select the permission called "SecretsManagerReadWrite" & "AmazonSSMManagedInstanceCore" and click next.
4) The name of the role will be: AppTier-Secrets-Role, and click on Create Role


## Part B) Create the Launch Template for the ASG to use.

>*Notes:* The launch template will help the ASG to create idential servers so it can provisioned the same blueprint of identical EC2 app servers.

1) Go to EC2 console, select Launch Templates on the left side of AWS, and click "Create Launch Templates" 
2) The name will be "app-tier-template"
3) For the Amazon Machine Image click "Quick Start", and select the "Amazon Linux 2023 AMI".
4) The instance type select "t2.micro".
5) For key pair select "Do not include in launch template". We will be using SSM to connect instead of SSH. SSM provide better security as it relies on identity-base IAM credentials. And SSM is centrally managed using IAM policies. 
6) For network settings, do not select a subnet and select the Security Group of the app tier (ec2-sg)
7) Go to the advanced details, click on IAM Instance Profile, and select the "AppTier-Secrets-Role"

## Part C) Create the Internal Target Group

> *Notes:* This is a list of the servers (in the app tier) that are healthy and ready to take traffic from the ALB. The internal Target Groups are importnat because it is the one that sends health checks to the servers in the app tier to see if they are healthy, and ready to receive traffic. When ASG provision a new instance, it will be automatically added to the target group, it is highly available as it will work between different AZs.

1) Go to the EC2 console, and on the left side click on "Target Groups" under Load Balancing 
2) Click "Create Target Group".
3) Select "Instances". The target group name "app-tier-tg". Protocol "HTTP" port 8080. For the VPC selects the "3-Tier-App-Project".
4) Set the Health check path as "/". This is the path where the Target Group will send the pings to make sure the instances are healthy
5) Leave the Register Targets app completly blank, and click on Create Target Group. 

## Part D) Create the Internal Application Load Balancer (ALB)

> **Notes:** The Application Load Balancer (ALB) sits inside the private subnet. It receives the incoming requets from the web tier, and distributes the traffic across the App Tier Instances.

1) On the EC2 menu, click "Load Balancers" on the left, and click "Application Load Balancer" and create.
2) Load balancer name: alb-internal-app. 
3) Scheme: Internal
4) Load Balancer IP address type: IPv4.
5) Network Mapping: Select the "3-Tier-App-Project" VPC. Check both Availability Zones (us-east-1a & us-east-1b). And select the AZA-AppTier-Private & AZB-AppTier-Private subnets.
6) Security Groups: Remove the default, and add: internal-alb-sg
7) For the listeners and routing, add HTTP port 80, and on the routing action, select the app-tier-tg

## Part E) Create the App Tier Auto Scaling Group (ASG).

> **Notes:** The ASG uses the Launch Template for the group (app-tier-template), the Target Group, and the private app tier subnet. It automatically scale out or in the EC2 instances across both AZ zones based on demand. The ASG will monitor the health, and performance of the EC2 instances, and make the decision if scale out or scale in. This monitor is different as the Target Group as the TG will check which instances are available to send the traffic from the private ALB, and the monitor of ASG is to see if it needs to scale out or in (add or remove instances)

1. Go to the EC2 menu, and on the left side click on Auto Scaling Group, and click on "Create Auto Scaling Group"
2. Name type "app-tier-asg". On the launch template select "app-tier-template", and click next. 
3. VPC select the "3-Tier-App-Project", and select both Availability Zones, and subnets that will be provision: AZA-AppTier-Private (us-east-1a) and AZB-AppTier-Private (us-east-1b). Leave the rest as default.
4. On the Load Balancing, click "Attach to an existing load balancer", then "choose from your load balancer target groups", and select the "app-tier-tg".
5. Turn on "Elastic Load Balancing health checks". The ELB will send health check to the EC2 instance, and if they do not pass the test. The ELB marks it as "unhealthy" so the ASG can read it, and replace the unhealthy EC2 instance.
6. Health check grace period: 300 seconds. Click next
7. Select the group size, and scaling policies. For desired capacity select 1, for minimun capacity select 1, and for maximun capacity select 3. This means that the Auto Scaling Group will always have one instances running at all time (desired capacity), and it will provisioned up to 3 instances (maximum capacity). The minimun capacity is equal to the desired capacity so it will always be running with one.
8. Select "Target tracking scaling policy". This will help us keep track of a metric so ASG can decide if it needs to provision or remove resources. The name will be "Target Tracking Policy: Ave CPU Utilization" Metric type will be "Average CPU Utilization", and target value will be "50". If the EC2 instance's Average CPU is equal or more than 50, then it will provision a another instance.

** Health Check Summary: ** It might seems like to many health check in the enviroment, but each make their own function within the systems.

A) EC2 Status Check: This is on the AWS infrastructure, it will check the physical host where the EC2 instances are, network connectivity, kernel, etc. If inside the applicaiton something crash, the EC2 status check will not know. 

B) Target Group Health Check: This is on the application layer. The Internal ALB sends HTTP GET requests to the port to see if the application is up, and working (HTTP 200 ok). If the ping failed, the Internal ALB is not able to provisioned more instances, just do not send traffic to the failing EC2s.

C) ASG Health Check (ELB + EC2): This is for the Lifecycle & Replacement. ASG asks both EC2 and the TG if the instances are healthy. 

D) The Target Tracking Policy is different, it will only check the resources on the EC2 instances, and determine if they need to provision or remove another one. 

## Part F) Verify the private subnet connectivity via AWS SSM Session Manager.

> **Note1:** We will not be connecting to our EC2 instances via SSH (Port 22). Instead, we will use AWS Systems Manager (SSM) Session Manager to check our network connectivity, private subnet routing, and IAM permissions.

> **Note2:** Because the App Tier is in a private subnet, it cannot connect to the public internet, which normally prevents SSM from working. This is where Interface Endpoints come in handy to provide extra security. Instead of enabling a NAT Gateway on the VPC, we create three VPC endpoints that establish a private, direct tunnel between our VPC and the AWS public services. AWS places a private network interface (with a local IP from our subnet) directly inside the VPC. The EC2 instance talks to that local IP, and AWS routes the traffic internally over its own backbone network.

To allow this, we must create three Interface Endpoints:

* SSM Endpoint: The core control channel.
* SSMMESSAGES Endpoint: Handles encryption and the two-way terminal output.
* EC2MESSAGES Endpoint: Handles Run Command requests.

I) Modify DNS settings on VPC.

1) Go to the VPC menu in AWS, and select the VPC "3-Tier-App-Project".
2) On the upper left corner, click on Actions, and Edit VPC settings. 
3) In the VPC settings, make sure the DNS settings: Enable DNS resolution & Enable DNS hostnames are both selected.

II) Create VPC endpoints.

1) In the same VPC menu, on the left click on "Endpoints".
2) In Endpoints, click on "Create".
3) Name the first Endpoint as "ssm-endpoint", leave the category as "AWS Services". Under services, find the "com.amazonaws.us-east-1.ssm". Then, select the VPC for the project, the two availability zones, and the two app-tier subnets. Make sure private DNS name is enabled. Finally, select the Security Group "vpce-sg", and create the endpoint.
4) Name the second Endpoint as "ssmmessages-endpoint", leave the category as "AWS Services". Under services, find the "com.amazonaws.us-east-1.ssmmessages". Then, select the VPC for the project, the two availability zones, and the two app-tier subnets. Make sure private DNS name is enabled. Finally, select the Security Group "vpce-sg", and create the endpoint.
5) Name the third Endpoint as "ec2messages-endpoint", leave the category as "AWS Services". Under services, find the "com.amazonaws.us-east-1.ec2messages". Then, select the VPC for the project, the two availability zones, and the two app-tier subnets. Make sure private DNS name is enabled. Finally, select the Security Group "vpce-sg", and create the endpoint.

III) Verify Connection

1) Go to EC2 console, and go to "instances".
2) The "app-tier-asg" should have created an EC2 instance. Make sure it is running.
3) CLick on the EC2 instance, and click "Connect"
4) Click on the Session Manager, and click on connect. 
5) If it does not connect, reboot the system and try to connect via SSM again. 




LESSONS LEARNED:

1. Naming convention in SG are important.
2. AWS services like RDS, Target Groups, ASG.
3. Attention to detail
4. Learn more about VPC Endpoint 
