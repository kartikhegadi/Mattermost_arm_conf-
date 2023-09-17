# Mattermost_arm_conf-
This instructions for arm (mac m1 ) architecture 


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
