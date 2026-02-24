# Nextcloud RDS Deployment

A production-ready deployment of [Nextcloud](https://nextcloud.com/) on AWS infrastructure, using an EC2 Ubuntu server with Nginx, PHP-FPM, and an Amazon RDS MySQL database backend.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Deployment Steps](#deployment-steps)
- [Database Setup](#database-setup)
- [Security Configuration](#security-configuration)
- [CRUD Testing](#crud-testing)
- [Screenshots](#screenshots)
- [How to Run](#how-to-run)
- [Author](#author)

---

## Project Overview

This project demonstrates deploying Nextcloud — a self-hosted file storage and collaboration platform — on AWS. The application is served from an EC2 Ubuntu instance using Nginx as the web server and PHP-FPM as the FastCGI process manager. All persistent data is stored in an Amazon RDS MySQL instance, which is hardened to accept connections only from the EC2 instance's security group.

Key goals:
- Self-hosted cloud storage accessible via the EC2 public IP address
- Separation of compute (EC2) and database (RDS) tiers for scalability and maintainability
- Minimal attack surface through tight security group rules

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   Internet                      │
└──────────────────────┬──────────────────────────┘
                       │  HTTP/HTTPS (port 80/443)
          ┌────────────▼────────────┐
          │    AWS EC2 (Ubuntu)     │
          │  ┌───────────────────┐  │
          │  │      Nginx        │  │  ← Web Server
          │  └────────┬──────────┘  │
          │           │ FastCGI     │
          │  ┌────────▼──────────┐  │
          │  │    PHP-FPM        │  │  ← Application Runtime
          │  └────────┬──────────┘  │
          │           │             │
          │  ┌────────▼──────────┐  │
          │  │   Nextcloud App   │  │  ← Application Files
          │  └───────────────────┘  │
          └────────────┬────────────┘
                       │  MySQL (port 3306)
                       │  (EC2 Security Group only)
          ┌────────────▼────────────┐
          │    Amazon RDS MySQL     │  ← Managed Database
          └─────────────────────────┘
```

**Traffic flow:**
1. Users access Nextcloud via the EC2 instance's public IP address on port 80 (or 443 with SSL).
2. Nginx handles incoming HTTP requests and passes PHP requests to PHP-FPM via FastCGI.
3. PHP-FPM executes the Nextcloud PHP application.
4. Nextcloud connects to the Amazon RDS MySQL instance over port 3306, which only allows inbound connections from the EC2 security group.

---

## Tech Stack

| Component        | Technology                          |
|------------------|-------------------------------------|
| Cloud Provider   | Amazon Web Services (AWS)           |
| Compute          | Amazon EC2 (Ubuntu 22.04 LTS)       |
| Web Server       | Nginx                               |
| Application      | Nextcloud (PHP)                     |
| PHP Runtime      | PHP-FPM                             |
| Database         | Amazon RDS (MySQL 8.x)              |
| Networking       | AWS VPC, Security Groups            |

---

## Deployment Steps

### 1. Launch an EC2 Instance

1. Sign in to the AWS Management Console and navigate to **EC2**.
2. Launch a new instance with **Ubuntu 22.04 LTS** as the AMI.
3. Choose an appropriate instance type (e.g., `t3.small` or larger).
4. Configure a security group to allow inbound traffic on port **22** (SSH) and port **80** (HTTP).
5. Attach or create a key pair for SSH access.

### 2. Connect to the Instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 3. Update the System and Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx php-fpm php-mysql php-xml php-mbstring \
    php-curl php-zip php-gd php-intl php-bcmath php-imagick unzip wget
```

### 4. Download and Configure Nextcloud

```bash
cd /var/www/
sudo wget https://download.nextcloud.com/server/releases/latest.zip
sudo unzip latest.zip
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

### 5. Configure Nginx

Create a new Nginx server block:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

Paste the following configuration (replace `<EC2_PUBLIC_IP>` with your EC2 public IP):

```nginx
server {
    listen 80;
    server_name <EC2_PUBLIC_IP>;

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Enable the configuration and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Database Setup

### 1. Launch an Amazon RDS Instance

1. Navigate to **RDS** in the AWS Console.
2. Click **Create database** and select **MySQL** (version 8.x).
3. Choose the **Free tier** template (or the appropriate tier for your workload).
4. Set a database identifier, master username, and master password.
5. Under **Connectivity**, select the same VPC as your EC2 instance.
6. Set **Public access** to **No**.
7. Under **VPC security groups**, create a new security group (or assign an existing one) that restricts MySQL access to the EC2 security group only (see [Security Configuration](#security-configuration)).

### 2. Create the Nextcloud Database and User

Connect to the RDS instance from your EC2 instance:

```bash
mysql -h <RDS_ENDPOINT> -u admin -p
```

Run the following SQL commands (replace `<STRONG_PASSWORD>` with a unique, randomly generated password — consider using a password manager or [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)):

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextclouduser'@'%' IDENTIFIED BY '<STRONG_PASSWORD>';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

### 3. Configure Nextcloud to Use RDS

Open a browser and navigate to `http://<EC2_PUBLIC_IP>`. Complete the Nextcloud setup wizard with the following database details:

| Field             | Value                          |
|-------------------|--------------------------------|
| Database host     | `<RDS_ENDPOINT>:3306`          |
| Database name     | `nextcloud`                    |
| Database user     | `nextclouduser`                |
| Database password | `<STRONG_PASSWORD>`            |

---

## Security Configuration

### EC2 Security Group (Inbound Rules)

| Type  | Protocol | Port | Source      | Purpose              |
|-------|----------|------|-------------|----------------------|
| SSH   | TCP      | 22   | Your IP     | Admin access         |
| HTTP  | TCP      | 80   | 0.0.0.0/0   | Web traffic          |
| HTTPS | TCP      | 443  | 0.0.0.0/0   | Secure web traffic   |

### RDS Security Group (Inbound Rules)

| Type         | Protocol | Port | Source               | Purpose              |
|--------------|----------|------|----------------------|----------------------|
| MySQL/Aurora | TCP      | 3306 | EC2 Security Group   | Database access only |

> **Important:** The RDS instance is not publicly accessible. Only the EC2 instance (via its security group) is allowed to connect to MySQL on port 3306. This prevents direct database access from the internet.

### Additional Hardening

- Store sensitive credentials in environment variables or AWS Secrets Manager rather than in plain-text config files.
- Enable HTTPS using a TLS certificate (e.g., Let's Encrypt / Certbot) for production deployments.
- Keep the EC2 instance and all packages up to date with regular `apt upgrade` runs.
- Enable AWS CloudTrail and VPC Flow Logs for auditing.

---

## CRUD Testing

The following tests verify that the Nextcloud application can correctly perform Create, Read, Update, and Delete operations against the RDS MySQL database.

### Create

1. Log in to Nextcloud at `http://<EC2_PUBLIC_IP>`.
2. Navigate to the **Files** section.
3. Click **New** → **New folder** and name it `test-folder`.
4. Upload a file (e.g., `test-document.txt`) into `test-folder`.
5. **Expected result:** The file and folder appear in the file list and are stored in the RDS database.

### Read

1. Refresh the Nextcloud Files page.
2. Verify that `test-folder` and `test-document.txt` are listed.
3. **Expected result:** Files are retrieved and displayed correctly from the database.

### Update

1. Right-click `test-document.txt` and select **Rename**.
2. Rename it to `updated-document.txt`.
3. **Expected result:** The new filename is reflected immediately and persisted in RDS.

### Delete

1. Right-click `updated-document.txt` and select **Delete**.
2. Confirm the deletion.
3. Navigate to the **Deleted files** section to confirm the item appears there.
4. Empty the trash to permanently remove the record.
5. **Expected result:** The file is removed from the UI and the corresponding database record is deleted.

---

## Screenshots

> Add screenshots below to document the deployment and test results.

### Nextcloud Login Page
![Nextcloud Login](screenshots/01-nextcloud-login.png)

### Nextcloud Dashboard
![Nextcloud Dashboard](screenshots/02-nextcloud-dashboard.png)

### RDS Instance in AWS Console
![RDS Instance](screenshots/03-rds-instance.png)

### EC2 Security Group Rules
![EC2 Security Group](screenshots/04-ec2-security-group.png)

### RDS Security Group Rules
![RDS Security Group](screenshots/05-rds-security-group.png)

### CRUD – File Upload (Create)
![File Upload](screenshots/06-crud-create.png)

### CRUD – File List (Read)
![File List](screenshots/07-crud-read.png)

### CRUD – File Rename (Update)
![File Rename](screenshots/08-crud-update.png)

### CRUD – File Delete
![File Delete](screenshots/09-crud-delete.png)

---

## How to Run

### Prerequisites

- An AWS account with permissions to create EC2 and RDS resources.
- An SSH key pair for EC2 access.
- A local machine with an SSH client and a web browser.

### Quick Start

1. **Provision infrastructure** — Launch an EC2 Ubuntu instance and an RDS MySQL instance in the same VPC following the steps in [Deployment Steps](#deployment-steps) and [Database Setup](#database-setup).
2. **Configure security groups** — Apply the rules described in [Security Configuration](#security-configuration).
3. **Deploy Nextcloud** — Install Nginx, PHP-FPM, and Nextcloud on the EC2 instance, then configure Nginx as described in [Deployment Steps](#deployment-steps).
4. **Connect Nextcloud to RDS** — Complete the Nextcloud web installer at `http://<EC2_PUBLIC_IP>` using the RDS endpoint and database credentials.
5. **Access the application** — Open `http://<EC2_PUBLIC_IP>` in your browser and log in with the admin credentials set during installation.

---

## Author

**Sinon1310**  
GitHub: [@Sinon1310](https://github.com/Sinon1310)
