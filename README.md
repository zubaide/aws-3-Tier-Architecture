# AWS 3-Tier Architecture Setup Guide

## Table of Contents
- [Overview](#overview)
- [Architecture Overview](#architecture-overview)
- [Step 1: VPC Setup](#step-1-vpc-setup)
- [Step 2: Security Configuration](#step-2-security-configuration)
- [Step 3: EC2 Instance Setup](#step-3-ec2-instance-setup)
- [Step 4: Database Setup](#step-4-database-setup)
- [Step 5: Testing Connectivity](#step-5-testing-connectivity)
- [Security Best Practices](#security-best-practices)
- [Contributing](#contributing)
- [License](#license)

## Overview
This guide provides step-by-step instructions for setting up a secure and scalable 3-tier architecture in AWS (Amazon Web Services). The architecture implements best practices for cloud infrastructure deployment with distinct presentation, application, and data tiers.

## Architecture Overview
Here's the complete architecture we'll be building:

![Tier3Topology](https://github.com/user-attachments/assets/8bbbd5ba-3a64-4a61-9435-5ee27b9849d9)

### Network Layout
- **Region:** US West (Oregon) / us-west-2
- **VPC:** 3-tier-VPC
- **Availability Zones:** us-west-2a, us-west-2b

### Subnet Structure
| Subnet Name | CIDR Block | Availability Zone | Type |
|-------------|------------|-------------------|------|
| Public Subnet 1 | 10.0.0.0/24 | us-west-2a | Public |
| Private Subnet 1 | 10.0.1.0/24 | us-west-2a | Private |
| Private Subnet 2 | 10.0.2.0/24 | us-west-2a | Private |
| Private Subnet 3 | 10.0.3.0/24 | us-west-2b | Private |

## Step 1: VPC Setup

### 1.1 Create VPC
Create your VPC with the following configuration:

![VPC](https://github.com/user-attachments/assets/ab273dbb-ab04-486f-86fc-9bfd05ea11a8)

### 1.2 Configure Subnets
Add subnets according to the structure defined above:

![Subnet](https://github.com/user-attachments/assets/0970b52b-eecf-48d1-a073-0a5e9d1585ff)

### 1.3 Set Up Internet Gateway
Create and attach the Internet Gateway (3-tier-IGW):

![Internet Gateway](https://github.com/user-attachments/assets/d49f541e-7c7b-4ea0-9002-a97a8ff7dd02)

### 1.4 Configure NAT Gateway
Create NAT Gateway (3-tier-natGW) in Public Subnet:

![NAT Gateway](https://github.com/user-attachments/assets/021ff2d1-4fdc-432c-a34f-a8a1efc525e0)

### 1.5 Set Up Route Tables
Configure your route tables as shown:

![Route Table](https://github.com/user-attachments/assets/d1ce8aa9-5508-4462-ba1e-a3c7f43e7fa6)

**Public Route Table (3-tier-public-RT):**
- Route to IGW (0.0.0.0/0)
- Associate with Public Subnet 1

**Private Route Table (3-tier-private-RT):**
- Route to NAT Gateway (0.0.0.0/0)
- Associate with all Private Subnets

## Step 2: Security Configuration

### 2.1 Create Security Groups
Set up your security groups with these configurations:

![Security Group](https://github.com/user-attachments/assets/b0a4e637-dde2-46cc-b17d-bc86f3cf47f6)

#### Bastion Host (bastion-host-SG)
```
Inbound Rules:
- HTTP (0.0.0.0/0)
- HTTPS (0.0.0.0/0)
- SSH (0.0.0.0/0)
- MYSQL/Aurora (database-server-SG)
```

#### Web Server (web-server-SG)
```
Inbound Rules:
- HTTP (0.0.0.0/0)
- HTTPS (0.0.0.0/0)
- SSH (0.0.0.0/0)
- All ICMP IPv4 (app-server-SG)
```

#### App Server (app-server-SG)
```
Inbound Rules:
- All ICMP IPv4 (web-server-SG)
- SSH (bastion-host-SG)
- MYSQL/Aurora (database-server-SG)
- HTTP (0.0.0.0/0)
- HTTPS (0.0.0.0/0)
```

#### Database Server (database-server-SG)
```
Inbound Rules:
- MYSQL/Aurora (app-server-SG)
- MYSQL/Aurora (bastion-host-SG)
```

## Step 3: EC2 Instance Setup

### 3.1 Launch EC2 Instances
Create your EC2 instances with these configurations:

![EC2 Instances](https://github.com/user-attachments/assets/19fb37dd-ef51-49de-b799-2ff065e68d6e)

#### Bastion Host
```bash
Configuration:
- AMI: Amazon Linux 2
- Instance Type: t2.micro
- VPC: 3-tier-VPC
- Subnet: Public Subnet 1
- Auto-assign Public IP: Enable
- Security Group: bastion-host-SG
```

#### Web Server
```bash
Configuration:
- AMI: Amazon Linux 2
- Instance Type: t2.micro
- VPC: 3-tier-VPC
- Subnet: Public Subnet 1
- Auto-assign Public IP: Enable
- Security Group: web-server-SG

User Data:
#!/bin/bash
sudo yum update -y
sudo amazon-linux-extras install -y lamp-mariadb10.11.9-php7.2 php7.2
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
```

#### App Server
```bash
Configuration:
- AMI: Amazon Linux 2
- Instance Type: t2.micro
- VPC: 3-tier-VPC
- Subnet: Private Subnet 1
- Auto-assign Public IP: Disable
- Security Group: app-server-SG

User Data:
#!/bin/bash
sudo yum install -y mariadb-server
sudo service mariadb start
```

## Step 4: Database Setup

### 4.1 Create DB Subnet Group
Set up your database subnet group:

![DB Subnet Groups](https://github.com/user-attachments/assets/c07b8230-0043-47c2-8ea8-af25a2c132e4)

```
Name: DBSubnetGroup
VPC: 3-tier-VPC
Subnets: Private Subnet 2, Private Subnet 3
```

### 4.2 Create MariaDB Instance
Configure your MariaDB instance:

![Database](https://github.com/user-attachments/assets/fd3309d0-066c-4ec7-91e5-c54c740cfbd9)

```
Engine: MariaDB 10.11.9
Template: Free tier
DB Instance Identifier: db-3-tier
Master Username: admin
Master Password: 3tierinfrastructure
VPC: 3-tier-VPC
Subnet Group: DBSubnetGroup
Public Access: No
```

## Step 5: Testing Connectivity

### 5.1 Connect to Bastion Host
First, SSH into your bastion host:

![SSH to Bastion Host](https://github.com/user-attachments/assets/88dc06e3-338e-4c98-a546-c3767dd65510)

```bash
# Download and set permissions for key pair
chmod 400 your-key-pair.pem
ssh -i "your-key-pair.pem" ec2-user@bastion-host-public-ip
```

### 5.2 Connect to App Server
From the bastion host, connect to your app server:

![SSH to App Server](https://github.com/user-attachments/assets/296b07a2-550c-4077-a124-c26e20873a9d)

```bash
# Copy key pair to bastion host and set permissions
chmod 400 your-key-pair.pem
ssh -i "your-key-pair.pem" ec2-user@app-server-private-ip
```

### 5.3 Test Network Connectivity
Verify connectivity between components:

![Ping to Web Server](https://github.com/user-attachments/assets/7cad52a7-3f04-420d-be1c-939ba3e35eee)

```bash
# From App Server
ping web-server-private-ip
mysql -h db-3-tier-endpoint -u admin -p
```

### 5.4 Verify Database Connection
Test the database connection:

![Screenshot 2024-10-16 162733](https://github.com/user-attachments/assets/1edeaa55-58e3-4072-827e-2b0d752afa85)

```bash
# From App Server
mysql --user="your_username" --password='your_password' --host="your_database_endpoint"
SHOW DATABASES;
```

## Security Best Practices

1. Regularly update security patches
2. Use AWS Secrets Manager for database credentials
3. Enable VPC Flow Logs for network monitoring
4. Implement AWS CloudWatch for monitoring
5. Use AWS CloudTrail for API auditing

## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License
[MIT](https://choosealicense.com/licenses/mit/)

---
**Note:** Replace placeholder values (IPs, endpoints, etc.) with your actual infrastructure values.
