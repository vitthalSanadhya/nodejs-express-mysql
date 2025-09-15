# Node.js + MySQL Deployment & Backup Automation Documentation

## 1. Introduction

### 1.1 Purpose
The purpose of this document is to describe in detail the process of automating both the deployment and the database backup for the Node.js Restful CRUD API with Express and MySQL application.

This project was completed using Bezkoder’s open-source GitHub repository as the base, with additional scripts and automation developed to meet the requirements of a 2-tier application deployment.

The document is intended for the following audiences:

- Developers, who may need to redeploy or modify the application.
- DevOps engineers, who are responsible for automating and maintaining the deployment and backup pipelines.
- System administrators, who will monitor backups, logs, and ensure infrastructure health.

It ensures that any future engineer can replicate, maintain, or improve the setup with minimal guidance.

### 1.2 Scope
This project specifically covers:

- **Deployment automation**: Using a shell script (`deploy.sh`) to pull the latest application code from GitHub, restart the Node.js application using PM2, and record the deployment details in a log file.
- **Database backup automation**: Using a shell script (`db_backup.sh`) to create a timestamped MySQL database dump, upload it securely to AWS S3, and log the operation.
- **Scheduling**: Using cron to automatically trigger the backup script every hour.

**Out of scope:**

- Enhancements to the frontend or user interface.
- Application feature development (e.g., adding or modifying CRUD operations).
- Implementation of high-availability or multi-region disaster recovery.

This ensures the focus remains solely on deployment and backup automation.

### 1.3 Definitions, Acronyms, and Abbreviations
- **CRUD**: Create, Read, Update, Delete – the four fundamental database operations.
- **PM2**: Process manager for Node.js applications.
- **AWS CLI**: Amazon Web Services Command Line Interface.
- **S3**: Simple Storage Service by AWS for storing backup files.
- **Cron**: Time-based job scheduler in Unix/Linux systems.

### 1.4 References
- **GitHub repository**: [vitthalSanadhya/nodejs-express-mysql](https://github.com/vitthalSanadhya/nodejs-express-mysql)
- **AWS CLI Documentation**: [AWS CLI Docs](https://docs.aws.amazon.com/cli/)
- **PM2 Documentation**: [PM2 Docs](https://pm2.keymetrics.io/)

### 1.5 Overview
This document is organized into the following sections:

1. Introduction  
2. Project Overview  
3. Functional Requirements  
4. Non-Functional Requirements  
5. Assumptions and Dependencies  
6. Risk Analysis  
7. Glossary  

---

## 2. Project Overview

### 2.1 Project Objectives
- Automate deployment of the Node.js + MySQL backend API.
- Enable automated, timestamped MySQL backups to AWS S3.
- Schedule backups using cron for hourly execution without manual intervention.

### 2.2 Business Goals
- **Efficiency**: Faster and reliable deployments.  
- **Data Protection**: Hourly backups reduce the risk of data loss.  
- **Auditability**: Logs for deployments and backups provide clear operational visibility.

### 2.3 Background
Based on Bezkoder’s Node.js + MySQL CRUD repository. Enhancements include:

- `deploy.sh` script to automate code deployment and PM2 restart.
- `db_backup.sh` script for MySQL backups and S3 upload.
- Cron jobs for hourly automated backups.

### 2.4 Constraints
- **AWS dependency**: Valid credentials required for S3 backups.  
- **Technology stack**: Server must have MySQL, PM2, and AWS CLI installed.

---

## 3. Functional Requirements

### 3.1 Use Case Diagram
```
[Developer] ---> [deploy.sh] ---> [Node.js App Restarted via PM2]
[System Cron] ---> [db_backup.sh] ---> [MySQL Backup] ---> [S3 Bucket]
```

### 3.2 Use Cases

#### 3.2.1 Use Case 1: Application Deployment
**Description:** Automates pulling the latest code and restarting the Node.js app.  
**Actors:** Developer, PM2  
**Preconditions:** EC2 instance running, Node.js installed  
**Postconditions:** Latest code deployed, logs updated  

**Flow of Events:**
1. Create a VPC (`10.0.0.0/16`) with a public subnet (`10.0.1.0/24`) connected to the Internet Gateway.  
2. Launch an Amazon Linux AMI instance in the public subnet.  
3. Install required components:
   - Node.js
   - PM2
   - AWS CLI
   - Cron
   - Git  
4. Run `deploy.sh` to pull the latest GitHub changes, install dependencies, and restart the Node.js app with PM2.  
5. Check deployment logs to confirm success.  
6. Verify the application by visiting `http://<Public-IP>:8080`.  

**Sample deploy.sh script:**
```bash
#!/bin/bash
# Deployment Script

APP_DIR="/home/ubuntu/nodejs-express-mysql"
APP_NAME="server"
LOG_FILE="/home/ubuntu/deploy.log"
REPO_URL="https://github.com/vitthalSanadhya/nodejs-express-mysql.git"
BRANCH="master"

echo "===== Deployment started: $(date) =====" >> "$LOG_FILE"

# Clone or pull repo
if [ ! -d "$APP_DIR" ]; then
    git clone -b "$BRANCH" "$REPO_URL" "$APP_DIR" >> "$LOG_FILE" 2>&1
else
    cd "$APP_DIR"
    git pull origin "$BRANCH" >> "$LOG_FILE" 2>&1
fi

# Install dependencies
cd "$APP_DIR"
npm install >> "$LOG_FILE" 2>&1

# Restart Node.js app
pm2 restart "$APP_NAME" >> "$LOG_FILE" 2>&1
echo "===== Deployment finished: $(date) =====" >> "$LOG_FILE"
```

---

#### 3.2.2 Use Case 2: Database Backup
**Description:** Automates MySQL backup with timestamp and uploads to S3.  
**Actors:** System cron, AWS CLI, MySQL  
**Preconditions:** Database configured, AWS credentials available  
**Postconditions:** Backup stored locally and uploaded to S3  

**Flow of Events:**
1. Install MongoDB and create a dedicated user.  
2. Install MySQL and secure installation:  
```bash
sudo apt update
sudo apt install mysql-server -y
sudo mysql_secure_installation
```
3. Create MySQL database and user:
```sql
CREATE DATABASE myappdb;
CREATE USER 'myappuser'@'localhost' IDENTIFIED BY 'mypassword';
GRANT ALL PRIVILEGES ON myappdb.* TO 'myappuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
4. Test MySQL user login:
```bash
mysql -u myappuser -p myappdb
```
5. Update `.env` file with database credentials:
```
DB_HOST=localhost
DB_USER=myappuser
DB_PASS=mypassword
DB_NAME=myappdb
```

**Database Backup Script (`db_backup.sh`):**
```bash
#!/bin/bash
DB_NAME="myappdb"
DB_USER="myappuser"
DB_PASS="Your_Password"
BACKUP_DIR="/home/ubuntu/db_backups"
TIMESTAMP=$(date +"%F-%H-%M-%S")
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_$TIMESTAMP.sql"
LOG_FILE="$BACKUP_DIR/db_backup.log"
S3_BUCKET="s3://myapp-db-backups-vs2001/db_backups"

mkdir -p $BACKUP_DIR
/usr/bin/mysqldump -u $DB_USER -p$DB_PASS $DB_NAME > $BACKUP_FILE 2>> $LOG_FILE

if [ $? -eq 0 ]; then
    echo "$(date) - Backup successful: $BACKUP_FILE" >> $LOG_FILE
    aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/" >> $LOG_FILE 2>&1
fi
```

---

## 4. Non-Functional Requirements

### 4.1 Performance Requirements
- Deployment should complete in <2 minutes.  
- Database backup should complete in <5 minutes for 1 GB data.  
- Cron jobs must execute exactly on schedule (hourly backups).

### 4.2 Security Requirements
- AWS keys stored securely in environment variables or AWS CLI config.  
- MySQL credentials must not appear in logs.  
- Cron jobs restricted to authorized users only.  
- Backups in S3 encrypted using AES-256 (SSE-S3).

### 4.3 Usability Requirements
- Deployment triggered with a single command: `./deploy.sh`.  
- Backup scheduling automated via cron.  
- Logs are human-readable and easily accessible.

### 4.4 Reliability Requirements
- S3 uploads must succeed with 99% reliability.  
- Failed uploads retried automatically via AWS CLI.  
- PM2 ensures application downtime <30 seconds during deployment.

### 4.5 Compliance Requirements
- Backups encrypted in S3.  
- Logs retained for audit purposes (≥30 days).  
- Backup retention policies implemented (older backups removed after 7–14 days).

---

## 5. Assumptions and Dependencies

### 5.1 Assumptions
- Server has internet access.  
- Valid GitHub and AWS credentials are available.  
- Required packages (MySQL, Node.js, PM2, AWS CLI) installed.

### 5.2 Dependencies
- AWS S3 availability.  
- MySQL service uptime.  
- PM2 installed for Node.js process management.  
- Cron installed for scheduling backups.

---

## 6. Risk Analysis

### 6.1 Risk Identification
- GitHub or network downtime affecting deployment.  
- MySQL service stopped causing backup failure.  
- Incorrect AWS credentials preventing S3 upload.  
- Misconfigured cron jobs causing missed backups.  
- Log files growing indefinitely.

### 6.2 Risk Mitigation
- Retry logic for git pull and S3 upload.  
- Check MySQL service before backup.  
- Validate AWS CLI configuration.  
- Regularly verify cron configuration.  
- Implement log rotation using logrotate.

---

## 7. Glossary
- **Deployment Script (`deploy.sh`)**: Automates code deployment and PM2 restart.  
- **Backup Script (`db_backup.sh`)**: Automates MySQL backups and S3 uploads.  
- **Cron**: Schedules recurring tasks (hourly backups).  
- **PM2**: Node.js process manager ensuring uptime.  
- **S3 Bucket**: AWS cloud storage for encrypted backups.
