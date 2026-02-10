# AWS EC2: Nginx + PHP‑FPM (PHP 8.3) End‑to‑End Lab

## Purpose
Deploy a full Nginx + PHP‑FPM stack on an Ubuntu EC2 instance, serve a PHP app, and verify FastCGI integration.

---

## 1. Launch EC2 Instance

### Instance settings
- **AMI:** Ubuntu Server 22.04 LTS  
- **Type:** t2.micro / t3.micro  
- **Network:** Public subnet + Auto‑assign public IP  
- **Security Group:**  
  - HTTP (80) → 0.0.0.0/0  
  - SSH (22) → your IP  
- **Key Pair:** RSA `.pem`

---

## 2. Connect to the Instance

```bash
chmod 400 php-fpm-lab-key.pem
ssh -i php-fpm-lab-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Update packages:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Test:

```bash
curl http://<EC2_PUBLIC_IP>
```

---

## 4. Install PHP‑FPM (PHP 8.3)

```bash
sudo apt install -y php-fpm php-cli php-mysql
```

Check installed socket:

```bash
ls /run/php/
```

Expected:

```
php-fpm.sock
php8.3-fpm.pid
php8.3-fpm.sock
```

Enable service:

```bash
sudo systemctl enable --now php8.3-fpm
```

---

## 5. Create Web Root

```bash
sudo mkdir -p /var/www/phpfpm-lab
sudo chown -R $USER:$USER /var/www/phpfpm-lab
```

Create test PHP file:

```bash
cat << 'EOF' > /var/www/phpfpm-lab/index.php
<?php phpinfo();
EOF
```

---

## 6. Create Nginx Server Block

Create config:

```bash
sudo nano /etc/nginx/sites-available/phpfpm-lab
```

Add:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/phpfpm-lab;
    index index.php index.html index.htm;

    access_log /var/log/nginx/phpfpm-lab.access.log;
    error_log  /var/log/nginx/phpfpm-lab.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/phpfpm-lab /etc/nginx/sites-enabled/
```

Disable default site:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Test + reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 7. Verify PHP‑FPM Integration

```bash
curl http://<EC2_PUBLIC_IP>
```

Expected: PHP info page rendered by PHP‑FPM.

---

## 8. Replace phpinfo (Optional Hardening)

```bash
rm /var/www/phpfpm-lab/index.php
cat << 'EOF' > /var/www/phpfpm-lab/index.php
<?php echo "Hello from Nginx + PHP-FPM on AWS!"; ?>
EOF
```

---

## 9. Optional: Create AMI

EC2 Console → Instance → **Actions → Image and templates → Create image**

---

## Reference: 
[How to Configure PHP-FPM with NGINX for Secure PHP Processing]([URL](https://www.digitalocean.com/community/tutorials/php-fpm-nginx))