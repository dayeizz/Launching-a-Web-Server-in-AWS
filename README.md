## Launching a Web Server in AWS
This guide provides step-by-step instructions for deploying a web server on Amazon Web Services (AWS). It covers selecting an EC2 instance, configuring security groups, installing web server software, and verifying that the application is accessible over the internet.

### 1. Start EC2 & Configure Security Group

1. Go to **EC2 → Instances → Locate your instance → Start Instance**
2. Go to **Security Groups → Locate your security group → Choose Edit inbound rules**

   * Add rule → Type: **All traffic**
   * Source: **Custom `<your_ip>/23`**. Copy your IP from here **https://ipchicken.com/**
   * Description: *your name*
  
   * Add rule → Type: **HTTP**
   * Source: **0.0.0.0**
   * Description: *http*
  
  * Add rule → Type: **HTTPS**
   * Source: **0.0.0.0**
   * Description: *https*

---
### 2. Configure Subdomains (Route 53)

1. Go to **Route 53 → Hosted Zones → example.com**
2. Create **A records** for:

   * `example.com`
   * `www.example.com`
   * `subdomain.example.com`
3. Point all to your server’s **public IP (EC2 → Instances → Locate your instance)**.

---

### 3. Connect via SSH

```bash
ssh -i privatekey.pem ubuntu@<public_ipv4>
```

---

### 4. Configure Apache

Edit Apache config:

```bash
sudo nano /etc/apache2/apache2.conf
```

Add at the end:

```apache
<Directory "/var/www/html">
    AllowOverride All
</Directory>
```

Create virtual host:

```bash
sudo nano /etc/apache2/sites-available/example.conf
```

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName example.com
    ServerAlias www.example.com subdomain.example.com
    DocumentRoot /var/www/html/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    RewriteEngine On
    RewriteCond %{SERVER_NAME} =example.com [OR]
    RewriteCond %{SERVER_NAME} =www.example.com [OR]
    RewriteCond %{SERVER_NAME} =subdomain.example.com
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

> 💡 **Note:** Ensure your `DocumentRoot` directory exists:
>
> ```bash
> sudo mkdir -p /var/www//html
> sudo chown -R www-data:www-data /var/www/html
> ```

Enable and reload:

```bash
sudo a2ensite example.conf
sudo systemctl reload apache2
sudo apache2ctl configtest
```

---

### 5. Install Certbot & Configure HTTPS

```bash
sudo add-apt-repository ppa:certbot/certbot -y
sudo apt update
sudo apt install python3-certbot-apache -y
sudo certbot --apache -d example.com -d www.example.com -d subdomain.example.com
```

Choose **Redirect** when prompted.
Certbot auto-renews every 90 days.

---

### 6. Secure Files with `.htaccess`

Create `/var/www/html/.htaccess`:

```apache
# Block directory listing
Options -Indexes

# Restrict sensitive files
<Files "config.php">
  Require all denied
</Files>

<FilesMatch "\.(ini|log|txt|bak|sql)$">
  Require all denied
</FilesMatch>

# Deny hidden files
<FilesMatch "^\.">
  Require all denied
</FilesMatch>

# Allow PHP execution
<FilesMatch "\.(php|phps|phtml)$">
  Require all granted
</FilesMatch>
```

---

### 7. File Transfer via SCP

```bash
scp -i privatekey.pem -r /local/folder ubuntu@<server_ip>:/var/www/html/
```

---

### 8. Troubleshooting

* **CSS not reloading:**
  Run `sudo chmod -R 755 /var/www/html` and hard-refresh (Ctrl + F5) on the browser.
* **Wrong redirect:**
  Check `ServerName`, `ServerAlias`, `RewriteRule` paths and hard-refresh (Ctrl + F5) on the browser.
* **Force reload config:**

  ```bash
  sudo systemctl reload apache2
  ```
