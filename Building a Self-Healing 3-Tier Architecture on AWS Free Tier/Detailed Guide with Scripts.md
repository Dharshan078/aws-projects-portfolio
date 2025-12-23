## Phase 1: Network and Security
  Created three Security Groups for Application Load Balancer (ALB), EC2 Instance Group and for RDS MySQL Database | ElastiCache
  1. Create ALB-SG for ALB inbound rule allows HTTP from internet
  2. Create WebServer-SG for EC2 Instance inbound rule allows HTTP traffic from ALB and SSH from Interent
  3. Create Database-SG for RDS and Elastic Cache inbound rule allows MySQL (3306) and Redis (6379) Traffic only from EC2 Instance

## Phase 2: Data Layer
  Create Amazon RDS (MySQL) and 
  1. Select MySQL as engine
  2. Use t3.micro for Instance
  3. Select a General purpose SSD with 20GB storage
  4. Turn off Public access and attach it to Database SG we previously Created
  Elastic Cache for Database and cache for Database offloading
  5. Select Redis cache
  
