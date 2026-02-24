# Deployment Commands

A reference of all terminal commands used during the Nextcloud RDS deployment.

---

## EC2 Setup

Update system packages and install required dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl unzip wget gnupg2 software-properties-common
```

Install PHP and required extensions:

```bash
sudo apt install -y php php-cli php-fpm php-mysql php-xml php-mbstring \
  php-curl php-zip php-gd php-intl php-bcmath php-gmp php-imagick
```

Download and extract Nextcloud:

```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
tar -xjf latest.tar.bz2 -C /var/www/html/
rm latest.tar.bz2
```

Configure PHP settings (replace `*` with your installed PHP version, e.g. `8.1`):

```bash
sudo nano /etc/php/*/fpm/php.ini
# Set: upload_max_filesize = 512M
# Set: post_max_size = 512M
# Set: memory_limit = 512M
# Set: max_execution_time = 360
```

---

## Web Server

Install and enable Apache:

```bash
sudo apt install -y apache2
sudo systemctl enable apache2
sudo systemctl start apache2
```

Enable required Apache modules:

```bash
sudo a2enmod rewrite headers env dir mime ssl
sudo systemctl restart apache2
```

Create an Apache virtual host for Nextcloud:

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

```apacheconf
<VirtualHost *:80>
    ServerName your-domain.com
    DocumentRoot /var/www/html/nextcloud

    <Directory /var/www/html/nextcloud>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Enable the site and reload Apache:

```bash
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

Obtain and configure an SSL certificate (Let's Encrypt):

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d your-domain.com
sudo systemctl reload apache2
```

---

## Database

Connect to the RDS instance:

```bash
mysql -h <rds-endpoint> -u <admin-user> -p
```

Create the Nextcloud database and user:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextclouduser'@'%' IDENTIFIED BY 'StrongPassword!';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

Verify connectivity from the EC2 instance to RDS:

```bash
mysql -h <rds-endpoint> -u nextclouduser -p nextcloud -e "SHOW TABLES;"
```

Install the MySQL client if not already present:

```bash
sudo apt install -y mysql-client
```

---

## Permissions

Set correct ownership for the Nextcloud directory:

```bash
sudo chown -R www-data:www-data /var/www/html/nextcloud
```

Set correct file and directory permissions:

```bash
sudo find /var/www/html/nextcloud -type f -exec chmod 640 {} \;
sudo find /var/www/html/nextcloud -type d -exec chmod 750 {} \;
```

Secure the Nextcloud config file after installation:

```bash
sudo chmod 600 /var/www/html/nextcloud/config/config.php
```

Set ownership for the Nextcloud data directory:

```bash
sudo mkdir -p /var/www/nextcloud-data
sudo chown -R www-data:www-data /var/www/nextcloud-data
sudo chmod 750 /var/www/nextcloud-data
```

---

## Troubleshooting

Check Apache error logs:

```bash
sudo tail -f /var/log/apache2/error.log
sudo tail -f /var/log/apache2/nextcloud_error.log
```

Check PHP-FPM logs (replace `*` with your installed PHP version, e.g. `8.1`):

```bash
sudo tail -f /var/log/php*/fpm/error.log
```

Check Nextcloud application logs:

```bash
sudo tail -f /var/www/nextcloud-data/nextcloud.log
```

Restart services after configuration changes (replace `*` with your installed PHP version, e.g. `8.1`):

```bash
sudo systemctl restart apache2
sudo systemctl restart php*-fpm
```

Run the Nextcloud maintenance commands (OCC):

```bash
sudo -u www-data php /var/www/html/nextcloud/occ status
sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --on
sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --off
sudo -u www-data php /var/www/html/nextcloud/occ upgrade
```

Check file integrity:

```bash
sudo -u www-data php /var/www/html/nextcloud/occ integrity:check-core
```

Test the database connection from the OCC tool:

```bash
sudo -u www-data php /var/www/html/nextcloud/occ db:convert-type --all-apps mysql nextclouduser <rds-endpoint> nextcloud
```

Verify open ports and connectivity:

```bash
sudo ss -tlnp | grep -E '80|443|3306'
nc -zv <rds-endpoint> 3306
```

Check EC2 instance security group and RDS security group rules:

```bash
aws ec2 describe-security-groups --group-ids <sg-id>
```
