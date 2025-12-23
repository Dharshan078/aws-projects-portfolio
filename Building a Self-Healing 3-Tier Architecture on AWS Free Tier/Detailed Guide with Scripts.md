## Phase 1: Network and Security
  Created three Security Groups for Application Load Balancer (ALB), EC2 Instance Group and for RDS MySQL Database | ElastiCache
  1. Create ALB-SG for ALB inbound rule allows HTTP from internet
  2. Create WebServer-SG for EC2 Instance inbound rule allows HTTP traffic from ALB and SSH from Interent
  3. Create Database-SG for RDS and Elastic Cache inbound rule allows MySQL (3306) and Redis (6379) Traffic only from EC2 Instance

## Phase 2: Data Layer
  Create Amazon RDS (MySQL)  
  1. Select MySQL as engine
  2. Use t3.micro for Instance
  3. Select a General purpose SSD with 20GB storage
  4. Turn off Public access and attach it to Database SG we previously Created

  Elastic Cache for Database and cache for Database offloading.  
  1. Select Redis cache
  2. Use Node-based Cluster
  3. For Configuration use Demo to save cost
  4. Connectivity select new subnet
  5. After creating Elasti Cache attach it to Database SG
  6. Notedown DNS Endpoint for RDS and ElastiCache
     RDS: wordpress-db.cin6eyikiuli.us-east-1.rds.amazonaws.com
     Redis: clustercfg.wordpress-cache.sbfkod.use1.cache.amazonaws.com:6379

## Phase 3: Compute Layer [EC2]
  We need to create a IAM Role for EC2 to talk with other AWS Services without the need to store secret key on Drive
  1. On IAM Dashboard Create a Role and Select EC2 for Service
  2. For Permissions Select AmazonSSMManagedInstanceCore and name it as EC2-Admin-Role

  Now create an EC2 Instance to build an Image and name it as Golden-Image-Builder or whatever you want
  1. Select t3.micro for EC2 Instance and for image select Amazon Linux 2023
  2. Select WebServer-SG we previously created on Phase 1 and for IAM Instance Policy Select EC2-Admin-Role
  3. Now for User Data
  `#!/bin/bash
  dnf update -y
  # Install Apache, PHP, and MariaDB (MySQL) client
  dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel mariadb105
  # Start and enable the Web Server
  systemctl start httpd
  systemctl enable httpd
  # Set permissions so we can edit files later
  usermod -a -G apache ec2-user
  chown -R ec2-user:apache /var/www
  chmod 2775 /var/www
  find /var/www -type d -exec chmod 2775 {} \;
  find /var/www -type f -exec chmod 0664 {} \;
  # Create a test file to verify PHP is working
  echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php`
  
