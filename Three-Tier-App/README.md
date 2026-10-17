### Secure & Scalable Three-Tier Application in AWS

## Overview
The objective of this project is to create a secure and scalable three-tier web application in AWS. The infrastructure is deployed in the **us-east-1** region within a custom Virtual Private Cloud (VPC) using the `192.168.0.0/24` CIDR block. To ensure high availability, the VPC spans two Availability Zones (**us-east-1a** and **us-east-1b**), each containing four distinct subnets.

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

## AWS Services Used

* **Amazon Route 53 (R53)**
* **Amazon CloudFront CDN + Web Application Firewall (WAF)**
* **Internet Gateway (IGW)**
* **Internal Application Load Balancer (ALB)**
* **Amazon EC2 (Web Tier):** 2 instances (Names and specs TBD)
* **Internal Application Load Balancer (ALB)**
* **Amazon EC2 (App Tier):** 2 instances (Names and specs TBD)
* **Amazon Aurora Database:** 1 Primary Database in AZ1
* **Amazon Aurora Database:** 1 Backup Read Replica in AZ2
* **Possible Considerations for Future Expansions:** Amazon S3, NAT Gateway, VPC Endpoints