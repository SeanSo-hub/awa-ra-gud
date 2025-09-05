# Awa sa ni, magamit ni nimo!

Use PuTTY for SSH and WinSCP for file transfers.

## Note — root workflow
This guide assumes you run commands as root (ssh root@your_server_ip). Running as root is convenient for setup but has security risks; consider creating a non-root sudo user after setup.

## Nano Shortcut keys
- Save = CTRL + O, then press Enter.
- EXIT = CTRL + X

## Quick commands
- restart apache = systemctl restart apache2
- restart mariadb = systemctl restart mariadb
- check your server basin luoy na kayo - htop

## 1. Connect to DigitalOcean Droplet
```bash
ssh root@your_server_ip
```

## 2. Update Server
```bash
apt update && apt upgrade -y
```

## 3. Install Apache
```bash
apt install -y apache2
systemctl status apache2
```
Open your browser to verify:
http://your-ip-address

## 4. Install MariaDB
```bash
apt install -y mariadb-server mariadb-client
mysql_secure_installation
```
Follow the interactive prompts (choose recommended answers). If MariaDB uses socket auth, you may not need a password for root.

or 

- Skip changing your password, if you have a strong password type "n"
- Remove anonymous users type "y"
- Next, disallow remote root login to prevent hackers from accessing your database type "n"
- Remove test database type "y"
- Finally, reload database type "y"

## 5. Install PHP (common modules for WordPress)
```bash
apt install -y php php-mysql php-xml php-mbstring php-curl php-gd php-zip unzip
```
Create a PHP info file to verify:
```bash
cat > /var/www/html/info.php <<'EOF'
<?php
phpinfo();
?>
EOF
```
Visit: http://your-ip-address/info.php

## 6. Create Database
```bash
mysql -u root -p
```
SQL (run inside mysql client):
```sql
CREATE DATABASE sample_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON sample_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 6.1 Import Database (optional)
```bash
mysql -u root -p sample_db < /var/www/html/your-folder/local.sql
```

## 7. Deploy WordPress to /var/www/html/your-folder
Upload and extract WordPress (example):
```bash
cd /var/www/html
unzip /path/to/wordpress.zip -d your-folder
```
Set ownership and permissions:
```bash
chown -R www-data:www-data /var/www/html/your-folder
find /var/www/html/your-folder -type d -exec chmod 755 {} \;
find /var/www/html/your-folder -type f -exec chmod 644 {} \;
```
Configure wp-config.php (edit with vim/nano or WinSCP):
```php
define( 'DB_NAME', 'sample_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'strong_password' );
define( 'DB_HOST', 'localhost' );
```

## 8. Update WordPress URLs (if migrating)
```bash
mysql -u root -p
USE sample_db;
UPDATE wp_options SET option_value = 'https://yourdomain.com' WHERE option_name IN ('siteurl','home');
UPDATE wp_posts SET post_content = REPLACE(post_content, 'http://old_ip/wp-content/uploads', 'https://yourdomain.com/wp-content/uploads');
EXIT;
```

## 9. DNS & SSL
- Add A records for @ and www pointing to your server IP.
- For SSL with certbot:
```bash
apt install -y certbot python3-certbot-apache
certbot --apache -d yourdomain.com -d www.yourdomain.com
systemctl restart apache2
```
If using Cloudflare, set SSL/TLS mode to Full (or Full (strict) if you have a proper cert).

## 10. Apache virtual host
Create site file:
```bash
cat > /etc/apache2/sites-available/yourdomain.conf <<'EOF'
<VirtualHost *:80>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    DocumentRoot /var/www/html/your-folder

    <Directory /var/www/html/your-folder>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
EOF
```
Enable and restart:
```bash
a2ensite yourdomain.conf
a2enmod rewrite
systemctl restart apache2
```

## 11. Post-setup recommendations
- Consider enabling UFW and only allowing necessary ports (22, 80, 443).
- Remove info.php after verifying PHP.
- Regularly back up the database and wp-content.
- After setup, create a non-root sudo user and disable root SSH if security is a priority.

## Troubleshooting & logs
- Apache: /var/log/apache2/error.log
- MariaDB: journalctl -u mariadb
- Common permission fix:
```bash
chown -R www-data:www-data /var/www/html/your-folder
```

[kung nag lisod ka anhi lang dri or e chatgpt, HAHA](https://www.digitalocean.com/community/tutorials/install-wordpress-on-ubuntu)