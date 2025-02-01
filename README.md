# AWS 3-Tier Architecture Setup Guide

## 📋 Overview

This guide provides step-by-step instructions for setting up a secure and scalable 3-tier architecture in AWS (Amazon Web Services). The architecture implements best practices for cloud infrastructure deployment with distinct presentation, application, and data tiers.

## 🏗️ Architecture Components

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

## 🚀 Setup Instructions

### 1. VPC Setup

1. Create VPC (3-tier-VPC)
2. Add Subnets according to the structure above
3. Create and attach Internet Gateway (3-tier-IGW)
4. Create NAT Gateway (3-tier-natGW) in Public Subnet
5. Configure Route Tables:
   ```
   Public Route Table (3-tier-public-RT):
   - Route to IGW (0.0.0.0/0)
   - Associate with Public Subnet 1

   Private Route Table (3-tier-private-RT):
   - Route to NAT Gateway (0.0.0.0/0)
   - Associate with all Private Subnets
   ```

### 2. Security Groups Configuration

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

### 3. EC2 Instance Setup

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

### 4. Database Setup (Amazon RDS)

1. Create DB Subnet Group:
   ```
   Name: DBSubnetGroup
   VPC: 3-tier-VPC
   Subnets: Private Subnet 2, Private Subnet 3
   ```

2. Create MariaDB Instance:
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

## 🔍 Testing Connectivity

1. SSH into Bastion Host:
   ```bash
   # Download and set permissions for key pair
   chmod 400 your-key-pair.pem
   ssh -i "your-key-pair.pem" ec2-user@bastion-host-public-ip
   ```

2. From Bastion Host to App Server:
   ```bash
   # Copy key pair to bastion host and set permissions
   chmod 400 your-key-pair.pem
   ssh -i "your-key-pair.pem" ec2-user@app-server-private-ip
   ```

3. Test Network Connectivity:
   ```bash
   # From App Server
   ping web-server-private-ip
   mysql -h db-3-tier-endpoint -u admin -p
   ```

4. Test Database Connectivity:
   ```bash
   # From App Server
   mysql --user="your_username" --password='your_database_password' --host="your_database_endpoint"
   SHOW DATABASES;
   ```

## ⚠️ Security Best Practices

1. Regularly update security patches
2. Use AWS Secrets Manager for database credentials
3. Enable VPC Flow Logs for network monitoring
4. Implement AWS CloudWatch for monitoring
5. Use AWS CloudTrail for API auditing

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## 📝 License

[MIT](https://choosealicense.com/licenses/mit/)

---

**Note:** Replace placeholder values (IPs, endpoints, etc.) with your actual infrastructure values.
