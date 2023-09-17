This instructions for arm **(mac m1** ) architecture 

# Mattermost 

A brief description of what this project does and who it's for

# Mattermost Installation Guide

This guide will walk you through the process of installing Mattermost on your server.

## Step 1: Download Mattermost

1. Visit the Mattermost release page on GitHub: [Mattermost Releases](https://github.com/SmartHoneybee/ubiquitous-memory/releases).
2. Download the Mattermost binary for your platform (e.g., mattermost-v7.5.2-linux-arm64.tar.gz.sha512sum).

```bash
tar -xvzf mattermost*.gz
sudo mv mattermost /opt
```

## Step 2 : Create Data Directory and User
```bash
sudo mkdir /opt/mattermost/data
sudo useradd --system --user-group mattermost
sudo chown -R mattermost:mattermost /opt/mattermost
sudo chmod -R g+w /opt/mattermost
````
## Step 4: Install MySQL (Optional: For PostgreSQL, follow this 
link https://jeffschering.github.io/mmdocs/upgrade/install/install-rhel-71.html)
```bash
sudo dnf update
sudo dnf install https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm
sudo dnf install mysql-server
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo mysql_secure_installation (optional)

```

## Step 5: Configure MySQL (Replace 'password' and '10.10.10.2' with appropriate values)
```bash
mysql -u root -p

>> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';

mysql> create user 'mmuser'@'%' identified by 'mmuser-password';
mysql> create database mattermost;
mysql> grant all privileges on mattermost.* to 'mmuser'@'%';

```

## Step 6: Configure Mattermost Database
Edit the Mattermost configuration file:
```bash
sudo vi /opt/mattermost/config/config.json

```
Update the following section:

```json
"DriverName": "mysql",
"DataSource": "mmuser:mmuser-password@tcp(localhost:3306)/mattermost?charset=utf8mb4,utf8&writeTimeout=30s"
```

## Step 7: Test Mattermost Installation
```bash
cd /opt/mattermost
sudo -u mattermost bin/mattermost

```
Access the Mattermost server at http://your-server-ip:8065 in your web browser.



## Step 8: Installing as a Service
Create a systemd service file:

```bash
sudo vi /lib/systemd/system/mattermost.service

```
Copy and paste the following configuration:

```bash 
[Unit]
Description=Mattermost
After=network.target
After=mysqld.service
BindsTo=mysqld.service

[Service]
Type=notify
ExecStart=/opt/mattermost/bin/mattermost
TimeoutStartSec=3600
KillMode=mixed
Restart=always
RestartSec=10
WorkingDirectory=/opt/mattermost
User=mattermost
Group=mattermost
LimitNOFILE=49152

[Install]
WantedBy=multi-user.target

```

## Step 9: Start Mattermost Service
```bash
sudo systemctl daemon-reload
sudo systemctl start mattermost.service
sudo systemctl enable mattermost.service

```

## Step 10: Setting up Reverse Proxy with Nginx


Create an Nginx configuration file: 
https://docs.mattermost.com/install/config-proxy-nginx.html

```bash
sudo vi /etc/nginx/conf.d/mattermost.conf
Copy and paste the Nginx configuration from this source.
```
Restart Nginx:

```bash
sudo systemctl restart nginx
```

## Step 11: Install SSL Certificate (Replace 'mattermost.example.com' with your domain)
```bash

sudo certbot --nginx -d mattermost.example.com

```
## Step 12: Verify Nginx Configuration
```bash

sudo nginx -t
```
If the configuration is correct, reload Nginx:

```bash 

sudo service nginx reload
```

Now visit your domain, your mattermost will be up and running.


```bash
```
Replace `mattermost.example.com` with your domain / subdomin pointing to your server.

```
upstream backend {
   server localhost:8065;
   keepalive 32;
}

proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mattermost_cache:10m max_size=3g inactive=120m use_temp_path=off;

server {
  listen 80;
  server_name   mattermost.example.com;
  return 301 https://$server_name$request_uri;
}

server {
   listen 443 ssl http2;
   server_name    mattermost.example.com;

   http2_push_preload on; # Enable HTTP/2 Server Push

   ssl on;
   ssl_certificate /etc/letsencrypt/live/mattermost.example.com/fullchain.pem;
   ssl_certificate_key /etc/letsencrypt/live/mattermost.example.com/privkey.pem;
   ssl_session_timeout 1d;

   # Enable TLS versions (TLSv1.3 is required upcoming HTTP/3 QUIC).
   ssl_protocols TLSv1.2 TLSv1.3;

   # Enable TLSv1.3's 0-RTT. Use $ssl_early_data when reverse proxying to
   # prevent replay attacks.
   #
   # @see: https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_early_data
   ssl_early_data on;

   ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384';
   ssl_prefer_server_ciphers on;
   ssl_session_cache shared:SSL:50m;
   # HSTS (ngx_http_headers_module is required) (15768000 seconds = six months)
   add_header Strict-Transport-Security max-age=15768000;
   # OCSP Stapling ---
   # fetch OCSP records from URL in ssl_certificate and cache them
   ssl_stapling on;
   ssl_stapling_verify on;

   add_header X-Early-Data $tls1_3_early_data;

   location ~ /api/v[0-9]+/(users/)?websocket$ {
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       client_max_body_size 50M;
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       client_body_timeout 60;
       send_timeout 300;
       lingering_timeout 5;
       proxy_connect_timeout 90;
       proxy_send_timeout 300;
       proxy_read_timeout 90s;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }

   location / {
       client_max_body_size 50M;
       proxy_set_header Connection "";
       proxy_set_header Host $http_host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       proxy_set_header X-Frame-Options SAMEORIGIN;
       proxy_buffers 256 16k;
       proxy_buffer_size 16k;
       proxy_read_timeout 600s;
       proxy_cache mattermost_cache;
       proxy_cache_revalidate on;
       proxy_cache_min_uses 2;
       proxy_cache_use_stale timeout;
       proxy_cache_lock on;
       proxy_http_version 1.1;
       proxy_pass http://backend;
   }
}

# This block is useful for debugging TLS v1.3. Please feel free to remove this
# and use the `$ssl_early_data` variable exposed by NGINX directly should you
# wish to do so.
map $ssl_early_data $tls1_3_early_data {
  "~." $ssl_early_data;
  default "";
}

```


Now test and reload the nginx....

```
sudo nginx -t
```

```
sudo service nginx reload
```

Now visit your domain, your mattermost will be up and running.
