# ☁️ AWS Project: Self-Healing 3-Tier Web Architecture

**Project Overview:**
Designed and deployed a highly available, fault-tolerant WordPress application on AWS. The infrastructure separates Compute (EC2) from Storage (RDS) and utilizes Caching (Redis) for performance, ensuring the application can survive availability zone failures and traffic spikes.

![Architecture Diagram](Screenshots/architecture-diagram.jpg)
*(Replace this path with your actual image file)*

---

## 🛠️ Phase 1: Network & Security (The Foundation)
We implement a "Security Group Chaining" strategy to ensure the Principle of Least Privilege.

1.  **ALB-SG (Load Balancer):**
    * **Inbound:** Allow HTTP (Port 80) from `0.0.0.0/0` (Internet).
2.  **WebServer-SG (EC2 Instances):**
    * **Inbound:** Allow HTTP (Port 80) **ONLY** from `ALB-SG`.
    * **Inbound:** Allow SSH (Port 22) **ONLY** from `My IP`.
3.  **Database-SG (Data Layer):**
    * **Inbound:** Allow MySQL (3306) **ONLY** from `WebServer-SG`.
    * **Inbound:** Allow Redis (6379) **ONLY** from `WebServer-SG`.

---

## 💾 Phase 2: The Data Layer
We provision the persistent storage and caching layer first to ensure they are ready for the application to connect.

**1. Amazon RDS (MySQL)**
* **Engine:** MySQL (Free Tier eligible).
* **Instance:** `db.t3.micro`.
* **Storage:** 20GB GP2 (General Purpose SSD).
* **Security:** **Public Access: NO**. Attach `Database-SG`.
* *Note:* Note down the **RDS Endpoint** (e.g., `wordpress-db.xxx.us-east-1.rds.amazonaws.com`).

**2. Amazon ElastiCache (Redis)**
* **Engine:** Redis (Cluster Mode Disabled for Free Tier).
* **Node Type:** `cache.t2.micro` (or `t3.micro`).
* **Subnet Group:** Create a new subnet group in private subnets.
* **Security:** Attach `Database-SG`.
* *Note:* Note down the **Primary Endpoint** (remove the `:6379` port number from the end string for later use).

---

## 💻 Phase 3: The Compute Layer (Golden Image)
We create a "Seed" instance to install software and verify connectivity before scaling.

**1. IAM Role (EC2-Admin-Role)**
* Create a Role with the policy: `AmazonSSMManagedInstanceCore`.
* *Why:* Allows secure connection to instances via AWS Systems Manager (SSM) without SSH keys.

**2. Launch EC2 Instance (Builder)**
* **OS:** Amazon Linux 2023.
* **Instance:** `t2.micro` / `t3.micro`.
* **Security:** Attach `WebServer-SG`.
* **IAM Role:** Attach `EC2-Admin-Role`.
* **User Data Script:** (Auto-installs LAMP Stack)

```bash
#!/bin/bash
dnf update -y

# Install Apache, PHP, and MariaDB (MySQL) client
dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel mariadb105

# Start and enable the Web Server
systemctl start httpd
systemctl enable httpd

# Set permissions for Apache
usermod -a -G apache ec2-user
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;

# Create test file
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
