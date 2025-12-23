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
  ```
#!/bin/bash
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
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```
  4. Launch Instance

  ## Phase 4: Connectivity Layer
  1. Now from created EC2 we should test that RDS and Redis is successfully connected or not
  2. To test for RDS
  ```mysql -h wordpress-db.cin6eyikiuli.us-east-1.rds.amazonaws.com -u admin -p```
  3. To test for Elasticache
  ```nc -zv clustercfg.wordpress-cache.sbfkod.use1.cache.amazonaws.com 6379```
  If nc is not installed use this command to install
  ```sudo dnf install nc -y```
  4. Login to SQL using this command
  ```mysql -h wordpress-db.cin6eyikiuli.us-east-1.rds.amazonaws.com -u admin -p```
  6. Create a database in MySQL
  ```CREATE DATABASE wordpress;```

  ## Phase 5: Application Layer
  1. Download and unzip wordpress
  ```
  cd /var/www/html
  sudo wget https://wordpress.org/latest.tar.gz
  sudo tar -xzf latest.tar.gz
  # Move files from the sub-folder to the main folder
  sudo cp -r wordpress/* .
  sudo rm -rf wordpress latest.tar.gz
```
  2. Configure the Database connection provided by wp-config-sample.php to wp-config.php
  ```sudo cp wp-config-sample.php wp-config.php```
  3. Now we need to add our database to wordpress
  ```sudo nano wp-config.php```
  ```
define( 'DB_NAME', 'database_name_here' ); -> Change to 'wordpress'

define( 'DB_USER', 'username_here' ); -> Change to 'admin'

define( 'DB_PASSWORD', 'password_here' ); -> Change to 'Password1234' (or whatever you set)

define( 'DB_HOST', 'localhost' ); -> Change to RDS Endpoint 'wordpress-db.cin6eyikiuli.us-east-1.rds.amazonaws.com'
  ```

  ## Phase 6: The "Unbreakable" Architecture
  This freezes the current EC2 as a working template
  1. Create a Image for the working EC2
  
  Create an Application load balancer for High availability
  1. Name it as WordpressALB
  2. Internet facing and attach it to ALB-SG we previously created on Phase-1
  3. For Listiners add HTTP Port:80 protocol
  4. Create a Target Group name it as Wordpress-TG and dont add EC2 Instance
  5. Create Load Balancer
  
  Launch a template for ASG
  1. Name it as Wordpress-LT
  2. Select AMI we created before
  3. Select Instance as t3.micro
  4. For Security Group attach it to WebServer-SG
  5. For IAM Role select: EC2AdminRole we previously created
  
  Create an Auto Scaling group for Scalability and Self-Healing
  1. Name it as Wordpress-ASG
  2. Select the Launch template we created before
  3. For Subnet select any two availability where the ASG will create two EC2 Instance
  4. Attach the existing ALB as Load Balancer and Create the ASG

  ## Phase 7: Testing
  Now you can view the application using ALB DNS address
  You can test by terminating the EC2 Intsance but ASG will automatically create another instance
  
