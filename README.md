# Awa sa ni, magamit ni nimo!

> Apache + PHP-FPM + MariaDB + Cloudflare SAFE  
> Tools: **PuTTY** for SSH · **WinSCP** for file transfers

---

## ⚠️ Important Rules (Read First)

- **NEVER** use Cloudflare Page Rules for domain redirects
- **NEVER** enable Cloudflare "Full (Strict)" before SSL is installed
- **ALWAYS** verify DNS before running Certbot
- **ALWAYS** ensure PHP-FPM is installed and linked to Apache
- Avoid multiple redirect sources (`.htaccess` + Cloudflare + WordPress)

---

## 🧠 Root Workflow

This guide assumes root access:

```bash
ssh root@your_server_ip
```

> **Nano shortcuts:** Save: `CTRL + O` → Enter · Exit: `CTRL + X`

---

## Step 1 — Update Server

```bash
apt update && apt upgrade -y
```

---

## Step 2 — Install Apache

```bash
apt install -y apache2
systemctl enable apache2
systemctl start apache2
```

Test by visiting `http://your-server-ip` in a browser.

---

## Step 3 — Install MariaDB

```bash
apt install -y mariadb-server mariadb-client
mysql_secure_installation
```

Recommended answers during setup:

- ✅ Set root password
- ✅ Remove anonymous users
- ✅ Disallow remote root login
- ✅ Remove test database
- ✅ Reload privileges

---

## Step 4 — Install PHP 8.2 + PHP-FPM ⚠️ IMPORTANT

```bash
apt install -y php8.2 php8.2-mysql php8.2-xml php8.2-mbstring \
  php8.2-curl php8.2-gd php8.2-zip unzip
apt install -y php8.2-fpm
systemctl enable php8.2-fpm
systemctl start php8.2-fpm
```

### Enable Apache PHP-FPM Bridge (CRITICAL)

```bash
a2enmod proxy_fcgi setenvif rewrite
a2enconf php8.2-fpm
systemctl restart apache2
```

---

## Step 5 — Create Database

```bash
mysql -u root -p
```

```sql
CREATE DATABASE sample_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON sample_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## Step 6 — Deploy WordPress

```bash
cd /var/www/html
unzip wordpress.zip -d your-folder
```

Set correct permissions:

```bash
chown -R www-data:www-data /var/www/html/your-folder
find /var/www/html/your-folder -type d -exec chmod 755 {} \;
find /var/www/html/your-folder -type f -exec chmod 644 {} \;
```

---

## Step 7 — Configure wp-config.php

```php
define('DB_NAME',     'sample_db');
define('DB_USER',     'wp_user');
define('DB_PASSWORD', 'strong_password');
define('DB_HOST',     'localhost');
```

---

## Step 8 — Apache Virtual Host

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

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF
```

Enable the site and disable the default:

```bash
a2ensite yourdomain.conf
a2dissite 000-default.conf
systemctl reload apache2
```

---

## Step 9 — DNS Check (CRITICAL — Do This Before SSL)

```bash
dig yourdomain.com +short
dig www.yourdomain.com +short
```

> ✅ Both must resolve to your server's IP address before proceeding.

---

## Step 10 — SSL Setup (Let's Encrypt)

Install Certbot:

```bash
apt install -y certbot python3-certbot-apache
```

**Before running Certbot, confirm:**
- Cloudflare SSL is set to **Full** (NOT Strict)
- DNS is pointing correctly
- No redirect rules to other domains

Run Certbot:

```bash
certbot --apache -d yourdomain.com -d www.yourdomain.com
```

When prompted, choose: **Redirect HTTP to HTTPS → YES**

---

## Step 11 — After SSL Success

In Cloudflare, change SSL/TLS mode to:

> **Full (Strict)** ✅

---

## Step 12 — Update WordPress URLs (HTTPS Fix)

```sql
UPDATE wp_options
SET option_value = 'https://yourdomain.com'
WHERE option_name IN ('siteurl', 'home');
```

---

## Step 13 — Troubleshooting

**Check service status:**

```bash
systemctl status apache2
systemctl status php8.2-fpm
```

**Check open ports:**

```bash
ss -tlnp | grep :80
ss -tlnp | grep :443
```

**Watch error logs:**

```bash
tail -f /var/log/apache2/error.log
```

---

## ⚡ Common Mistakes to Avoid

- ❌ **Cloudflare Page Rules for redirects**
  - 💥 Conflicts with Apache/WordPress redirects
  - ✅ Use `.htaccess` or WordPress settings only

- ❌ **Enabling Full (Strict) before SSL install**
  - 💥 SSL handshake errors
  - ✅ Set to **Full** first, switch to **Full (Strict)** after Certbot

- ❌ **Missing PHP-FPM bridge (`a2enconf php8.2-fpm`)**
  - 💥 PHP pages won't render
  - ✅ Run `a2enconf php8.2-fpm` then restart Apache

- ❌ **Wrong WordPress URL (http vs https mismatch)**
  - 💥 Redirect loops
  - ✅ Update `siteurl` and `home` in `wp_options` via SQL

- ❌ **Multiple redirect layers (.htaccess + WP + Cloudflare)**
  - 💥 Infinite redirect loops
  - ✅ Pick one redirect source and disable the rest

---

## ✅ Final Result (After Correct Setup)

- [x] Apache running
- [x] PHP-FPM connected
- [x] MariaDB running
- [x] SSL active (Let's Encrypt)
- [x] Cloudflare Full (Strict) enabled
- [x] No redirect loops
- [x] Stable WordPress deployment

[kung nag lisod ka anhi lang dri or e chatgpt, HAHA](https://www.digitalocean.com/community/tutorials/install-wordpress-on-ubuntu)
