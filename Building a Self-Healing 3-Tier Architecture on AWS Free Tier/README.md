# Building a Self-Healing 3-Tier Architecture on AWS Free Tier
## Objective: This project demonstrates a production-grade, fault-tolerant web application architecture on AWS. The goal was to deploy a **WordPress** application that is "self-healing"—capable of surviving server failures and traffic spikes without downtime. By decoupling the data layer (RDS + ElastiCache) from the compute layer (EC2 + Auto Scaling), the application achieves high availability and statelessness.

## 🛠️ Tech Stack & Services
* **Computing:** EC2 Instance, Auto Scaling Group [ASG]
* **Networking:** VPC, Security Groups, ELB: Application Load Balancer [ALB]
* **Database:** Amazon RDS (MySQL) for Persistent storage to save data from Wordpress
* **Cache:** Elasti Cache (Redis) for Session Management and Database Offloading
* **Security:** IAM Roles (SSM Access) for EC2 Instances, Security Group Chaining

## 🏗️ Implementation Steps

### Phase 1: Networking & Security
* Created a tiered Security Group structure to enforce the **Principle of Least Privilege**:
    * `ALB-SG`: Open to World (HTTP/80).
    * `App-SG`: Allows traffic **only** from `ALB-SG`.
    * `Data-SG`: Allows traffic **only** from `App-SG` on ports 3306 (MySQL) and 6379 (Redis).

### Phase 2: Data Layer Provisioning
* Deployed **RDS MySQL** (Free Tier) in a private subnet context.
* Deployed **ElastiCache (Redis)** cluster for caching frequently accessed data.
* *Verification:* Used `nc` (Netcat) and `mysql` CLI from a bastion host to verify connectivity.

### Phase 3: The Golden Image
* Configured an EC2 instance with Apache, PHP, and the MySQL connector.
* Integrated WordPress with the remote RDS database and Redis endpoint.
* Created an **AMI (Amazon Machine Image)** to serve as the blueprint for scaling.

### Phase 4: High Availability (The "Unbreakable" Layer)
* Configured an **Application Load Balancer** to distribute traffic across Availability Zones.
* Created a **Launch Template** using the custom AMI.
* Deployed an **Auto Scaling Group (ASG)** with Dynamic Scaling policies and ELB Health Checks.

## 🧪 Testing & Chaos Engineering
To verify resilience, I performed the following "Chaos" tests:

1.  **Traffic Test:** Verified the ALB distributes traffic using the DNS endpoint.
2.  **Persistence Test:** Created a blog post, then terminated the active EC2 instance.
    * *Result:* The new instance launched by ASG retained the data (content served from RDS).
3.  **Self-Healing Test:**
    * Simulated a crash by terminating a running instance.
    * **Result:** ASG detected the health check failure and provisioned a replacement within 120 seconds.

![ASG Activity History](screenshots/chaos-test-proof.png)

## 🚧 Challenges Faced
**Issue: WordPress "Error Establishing Database Connection"**
* **Problem:** Even though RDS connectivity was verified via CLI, WordPress failed to connect.
* **Diagnosis:** The `wp-config.php` file was pointing to the **DB Instance Identifier** (`wordpress-db`) instead of the actual **DB Name** (`wordpress`).
* **Solution:** Corrected the `DB_NAME` constant in the configuration file and manually verified the database creation in MySQL.

**Issue: Redis Connection Failure**
* **Problem:** The `nc` command failed to connect to the ElastiCache endpoint.
* **Solution:** Identified a syntax error where the port was appended with a colon (`:6379`) instead of a space. Also ensured the `dnf install nc` command was run on the minimal Amazon Linux 2023 OS.

## 🚀 Future Improvements
* **Infrastructure as Code:** Migrate the manual console setup to **Terraform** or **CloudFormation**.
* **HTTPS/SSL:** Attach an ACM Certificate to the Load Balancer for secure traffic.
* **Content Delivery:** Integrate **Amazon CloudFront** to cache static assets (images/CSS) at the edge.
