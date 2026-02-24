# Nextcloud on EC2 with RDS MySQL — Deployment Guide

This guide walks through deploying Nextcloud on an Amazon EC2 instance (Ubuntu 22.04 LTS) connected to an Amazon RDS MySQL database.

---

## Prerequisites

- An EC2 instance (Ubuntu 22.04 LTS, `t3.medium` or larger recommended)
- An RDS MySQL 8.0 instance (in the same VPC as the EC2 instance)
- Security groups configured to allow:
  - Port `80` / `443` inbound to the EC2 instance
  - Port `3306` from the EC2 instance's security group to the RDS security group
- SSH access to the EC2 instance
- A domain name (optional but recommended for HTTPS)

---

## 1. System Setup

Connect to your EC2 instance and update all packages.

```bash
ssh -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl unzip wget gnupg2 software-properties-common apt-transport-https ca-certificates lsb-release
```

---

## 2. PHP Installation

Nextcloud requires PHP 8.1 or higher. Install PHP and all required extensions.

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update

sudo apt install -y \
  php8.1 \
  php8.1-fpm \
  php8.1-cli \
  php8.1-mysql \
  php8.1-xml \
  php8.1-curl \
  php8.1-gd \
  php8.1-mbstring \
  php8.1-zip \
  php8.1-intl \
  php8.1-bcmath \
  php8.1-gmp \
  php8.1-imagick \
  php8.1-apcu \
  php8.1-redis
```

Verify the installation:

```bash
php -v
```

---

## 3. Nginx Installation & Configuration

### 3.1 Install Nginx

```bash
sudo apt install -y nginx
```

### 3.2 Download and Extract Nextcloud

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
sudo mv nextcloud /var/www/
```

### 3.3 Create the Nginx Server Block

Replace `your-domain.com` with your actual domain or EC2 public IP.

```bash
sudo tee /etc/nginx/sites-available/nextcloud <<'EOF'
upstream php-handler {
    server unix:/var/run/php/php8.1-fpm.sock;
}

server {
    listen 80;
    server_name your-domain.com;

    # Enforce HTTPS (uncomment after obtaining an SSL certificate)
    # return 301 https://$host$request_uri;

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json
               application/ld+json application/manifest+json application/rss+xml
               application/vnd.geo+json application/vnd.ms-fontobject application/wasm
               application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml
               application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest
               text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component
               text/x-cross-domain-policy;

    add_header Referrer-Policy                   "no-referrer" always;
    add_header X-Content-Type-Options            "nosniff" always;
    add_header X-Frame-Options                   "SAMEORIGIN" always;
    add_header X-Permitted-Cross-Domain-Policies "none" always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;
    add_header X-XSS-Protection                  "1; mode=block" always;

    fastcgi_hide_header X-Powered-By;

    include mime.types;
    types {
        text/javascript js mjs;
    }

    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location ^~ /.well-known {
        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }
        location /.well-known/acme-challenge { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation { try_files $uri $uri/ =404; }
        return 301 /index.php$request_uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/) { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    location ~ \.php(?:$|/) {
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS $https;

        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
        fastcgi_read_timeout 300;
    }

    location ~ \.(?:css|js|mjs|svg|gif|png|jpg|ico|wasm|tflite|map|ogg|flac)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;
        access_log off;
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;
        access_log off;
    }

    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
EOF
```

### 3.4 Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
```

---

## 4. Database Setup (RDS MySQL)

### 4.1 Install the MySQL Client

```bash
sudo apt install -y mysql-client
```

### 4.2 Connect to the RDS Instance

Replace the placeholders with your actual RDS values.

```bash
mysql -h <RDS_ENDPOINT> -u <RDS_MASTER_USER> -p
```

### 4.3 Create the Nextcloud Database and User

Run the following SQL commands inside the MySQL prompt:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

CREATE USER 'nextclouduser'@'%' IDENTIFIED BY '<DB_PASSWORD>';

GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';

FLUSH PRIVILEGES;

EXIT;
```

> **Security note:** Replace `<DB_PASSWORD>` with a strong, randomly generated password.
> You can generate one with: `openssl rand -base64 32`
> Store the password in AWS Secrets Manager or SSM Parameter Store — never hard-code it in scripts.

---

## 5. PHP-FPM Configuration

Tune PHP-FPM and `php.ini` for Nextcloud.

```bash
sudo sed -i 's/^memory_limit = .*/memory_limit = 512M/'       /etc/php/8.1/fpm/php.ini
sudo sed -i 's/^upload_max_filesize = .*/upload_max_filesize = 512M/' /etc/php/8.1/fpm/php.ini
sudo sed -i 's/^post_max_size = .*/post_max_size = 512M/'       /etc/php/8.1/fpm/php.ini
sudo sed -i 's/^max_execution_time = .*/max_execution_time = 300/' /etc/php/8.1/fpm/php.ini
sudo sed -i 's/^;date.timezone.*/date.timezone = UTC/'           /etc/php/8.1/fpm/php.ini
```

Enable OPcache:

```bash
sudo tee -a /etc/php/8.1/fpm/php.ini <<'EOF'

; Nextcloud OPcache settings
opcache.enable=1
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
EOF
```

---

## 6. File Permissions

Set the correct ownership and permissions for the Nextcloud directory.

```bash
sudo chown -R www-data:www-data /var/www/nextcloud
sudo find /var/www/nextcloud -type d -exec chmod 750 {} \;
sudo find /var/www/nextcloud -type f -exec chmod 640 {} \;
```

Create and set permissions for the Nextcloud data directory (store outside the web root for security):

```bash
sudo mkdir -p /var/nextcloud-data
sudo chown -R www-data:www-data /var/nextcloud-data
sudo chmod 750 /var/nextcloud-data
```

---

## 7. Nextcloud Installation via CLI

Run the Nextcloud installer using `occ` to avoid browser-based setup.

```bash
# Retrieve credentials from environment variables or AWS Secrets Manager.
# Never pass passwords directly on the command line as they can appear in
# shell history and process listings.
#
# Example: export DB_PASS=$(aws secretsmanager get-secret-value \
#   --secret-id nextcloud/db-password --query SecretString --output text)
#
# Or interactively:
read -rsp "DB password: "    DB_PASS   && echo
read -rsp "Admin password: " ADMIN_PASS && echo

sudo -u www-data php /var/www/nextcloud/occ maintenance:install \
  --database        "mysql" \
  --database-host   "<RDS_ENDPOINT>" \
  --database-name   "nextcloud" \
  --database-user   "nextclouduser" \
  --database-pass   "${DB_PASS}" \
  --admin-user      "admin" \
  --admin-pass      "${ADMIN_PASS}" \
  --data-dir        "/var/nextcloud-data"
```

Add your domain or IP to the list of trusted domains:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 0 --value="your-domain.com"
sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 1 --value="<EC2_PUBLIC_IP>"
```

---

## 8. Restart Services

Apply all configuration changes by restarting the relevant services.

```bash
sudo systemctl restart php8.1-fpm
sudo systemctl restart nginx

sudo systemctl enable php8.1-fpm
sudo systemctl enable nginx
```

Verify both services are running:

```bash
sudo systemctl status php8.1-fpm --no-pager
sudo systemctl status nginx --no-pager
```

---

## 9. Post-Installation Checks

Run the Nextcloud built-in check to confirm the setup is healthy:

```bash
sudo -u www-data php /var/www/nextcloud/occ status
sudo -u www-data php /var/www/nextcloud/occ check
```

Set up the background job cron:

```bash
sudo -u www-data php /var/www/nextcloud/occ background:cron
(crontab -l -u www-data 2>/dev/null; echo "*/5 * * * * php -f /var/www/nextcloud/cron.php --define apc.enable_cli=1") | sudo crontab -u www-data -
```

---

## Summary of Services

| Service       | Purpose                     | Port |
|---------------|-----------------------------|------|
| Nginx         | Web server / reverse proxy  | 80   |
| PHP-FPM 8.1   | PHP FastCGI process manager | —    |
| RDS MySQL 8.0 | Managed relational database | 3306 |

---

## Troubleshooting

- **Nginx config test fails:** Run `sudo nginx -t` and check `/var/log/nginx/error.log`.
- **PHP-FPM not starting:** Check `/var/log/php8.1-fpm.log` and verify the socket path in the Nginx upstream block.
- **Cannot connect to RDS:** Confirm the EC2 security group is added as an inbound source on port `3306` in the RDS security group.
- **Permission denied errors:** Re-run the `chown`/`chmod` commands in Section 6.
