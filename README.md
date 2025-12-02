## Launching a Web Server in AWS
This guide provides step-by-step instructions for deploying a web server on Amazon Web Services (AWS). It covers selecting an EC2 instance, configuring security groups, installing web server software, and verifying that the application is accessible over the internet.

---

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

### 2. Connect Domain to AWS

1. Open **Route 53 → Hosted Zones**
2. Click your domain
3.  Copy the **4 name servers** shown in the NS record
  (they look like: `ns-123.awsdns-45.com`)
4. Log in **Squarespace** → **Settings → Domains**
5. Select your domain
6. Open **Nameservers**
7. Choose **Use custom nameservers**
8. Delete the old nameservers
9. Paste the **4 AWS name servers**
10. Save
  
---

### 3. Configure Subdomains (Route 53)

1. Go to **Route 53 → Hosted Zones → example.com**
2. Create **A records** for:

   * `example.com`
   * `www.example.com`
   * `subdomain.example.com`
3. Point all to your server’s **public IP (EC2 → Instances → Locate your instance)**.
4. Go to: **[https://toolbox.googleapps.com/apps/dig/](https://toolbox.googleapps.com/apps/dig/)**
5. Type: **example.com**
6. Select: **A**
7. Click **Dig**
  You should see your EC2 public IP.

---

### 4. Connect via SSH

```bash
ssh -i privatekey.pem ubuntu@<public_ipv4>
```

---

### 5. Configure Apache

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

### 6. Install Certbot & Configure HTTPS

```bash
sudo add-apt-repository ppa:certbot/certbot -y
sudo apt update
sudo apt install python3-certbot-apache -y
sudo certbot --apache -d example.com
```

Choose **Redirect** when prompted.
Certbot auto-renews every 90 days.

---

### 7. Secure Files with `.htaccess`

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

### 8. File Transfer via SCP

```bash
scp -i privatekey.pem -r /local/folder ubuntu@<server_ip>:/var/www/html/
```

---

### 9. Troubleshooting

* **Unable to access SSH:**
Check your public IP (you can use a site like ipchicken.com).
Then go to Security Groups → Select your security group → Edit inbound rules, and make sure the rule for your name includes your current IP address.

* **CSS not reloading:**
  Run `sudo chmod -R 755 /var/www/html` and hard-refresh (Ctrl + F5) on the browser.
* **Wrong redirect:**
  Check `ServerName`, `ServerAlias`, `RewriteRule` paths and hard-refresh (Ctrl + F5) on the browser.
* **Force reload config:**

  ```bash
  sudo systemctl reload apache2
  ```
