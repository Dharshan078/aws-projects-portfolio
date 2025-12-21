# Building a Self-Healing 3-Tier Architecture on AWS Free Tier
## Objective: This project demonstrates a production-grade, fault-tolerant web application architecture on AWS. The goal was to deploy a **WordPress** application that is "self-healing"—capable of surviving server failures and traffic spikes without downtime. By decoupling the data layer (RDS + ElastiCache) from the compute layer (EC2 + Auto Scaling), the application achieves high availability and statelessness.

## 🛠️ Tech Stack & Services
* **Computing:** EC2 Instance, Auto Scaling Group [ASG]
* **Networking:** VPC, Security Groups, ELB: Application Load Balancer [ALB]
* **Database:** Amazon RDS (MySQL) for Persistent storage to save data from Wordpress
* **Cache:** Elasti Cache (Redis) for Session Management and Database Offloading
* **Security:** IAM Roles (SSM Access) for EC2 Instances, Security Group Chaining

