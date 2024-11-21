# Django Deployment Guide for Oracle Cloud Infrastructure (OCI)

This guide walks through deploying a Django application on Oracle Cloud Infrastructure using Ubuntu, Gunicorn, and Nginx.

## Prerequisites

- An Oracle Cloud Infrastructure (OCI) account
- A Django application ready for deployment
- Ubuntu VM set up on OCI
- Your DNS (added to cloudflare)

## 1. Create Virtual Cloud Network (VCN)

1. **Login to OCI Console**: Go to the Oracle Cloud Infrastructure console and log in with your credentials.
2. **Navigate to Networking**: In the main menu, under the "Core Infrastructure" section, click on "Networking" and then "Virtual Cloud Networks".
3. **Create VCN**: Click on "Create VCN".
![VCN Creation](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-1.png?raw=true)
4. **VCN Details**:
    - **Name**: Enter a name for your VCN.
    - **Compartment**: Create in default compartment, where your VM will reside.
    - **CIDR Block**: Enter a CIDR block for your VCN (e.g., 10.0.0.0/16).
    - **DNS Label**: Optionally, enter a DNS label for the VCN.
    ![VCN Wizard](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-2.png?raw=true?)
6. **Create Subnets**:
    - **Public Subnet**: Create a public subnet for resources that need to be accessible from the internet.
    - **Private Subnet**: Create a private subnet for internal resources.
7. **Internet Gateway**: Create an Internet Gateway to allow traffic to and from the internet.
8. **Route Table**: Configure route tables to define how traffic should be routed within the VCN.
9. **Security Lists**: Set up security lists to control the traffic allowed to and from the subnets.
10. **Review and Create**: Review your configuration and click "Create VCN".

Your Virtual Cloud Network is now set up and ready for use.

## 1. Initial Server Setup

### Connect to Server

```bash
ssh ubuntu@[ip_address] -i [path_to_private_key]
```

### Install Required Packages

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install python3-venv
```

## 2. Django Application Setup

### Use your own repo
If you use your own private repo create an ssh key on the server and copy into github allowed keys. You should probably disable push for this key.
```bash
git clone https://github.com/[your-repo]/[your-app].git
```

### Alternative: Transfer Local Files

```bash
scp -r [local_path] ubuntu@[IP]:[remote_path]
```

### Create and Activate Virtual Environment

```bash
python3 -m venv /home/ubuntu/[project_name]/venv
source /home/ubuntu/[project_name]/venv/bin/activate
```

### Install Dependencies

```bash
pip install gunicorn
```

## 3. Firewall Configuration

### Configure Ubuntu Firewall

```bash
sudo apt-get install iptables-persistent
sudo iptables -I INPUT 5 -p tcp --dport 8000 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
```

### Configure OCI Firewall

1. Navigate to OCI Console
2. Add ingress rule to security list for subnet
3. Allow port 80, 443

## 4. Gunicorn Setup

### Test Gunicorn
```bash
cd /home/ubuntu/[project_name]/[project_name]
gunicorn --bind 0.0.0.0:8000 [project_name].wsgi
```

### Create Gunicorn System Files

1. Create log directory:
```bash
sudo mkdir /var/log/gunicorn
```

2. Create socket file:

```bash
sudo nano /etc/systemd/system/gunicorn.socket
```

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

3. Create service file:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

```ini
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/[project_name]/[project_name]
ExecStart=/home/ubuntu/[project_name]/venv/bin/gunicorn \
    --access-logfile - \
    --workers 3 \
    --bind unix:/run/gunicorn.sock \
    [project_name].wsgi:application

[Install]
WantedBy=multi-user.target
```

### Start Gunicorn

```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

## 5. Nginx Setup

### Install Nginx

```bash
sudo apt-get install nginx
```

### Configure Django Settings

Update `settings.py`:
```python
DEBUG = False
ALLOWED_HOSTS = ['*']
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

### Collect Static Files

```bash
source /home/ubuntu/[project_name]/venv/bin/activate
python manage.py collectstatic
```

### Configure Nginx

1. Create site configuration:
```bash
sudo nano /etc/nginx/sites-available/[project_name]
```
```nginx
server {
    listen 80;
    server_name [your_ip_address];

    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }
    
    location /static/ {
        root /home/ubuntu/[project_name]/[project_name];
    }
    
    location /media/ {
        root /home/ubuntu/[project_name]/[project_name];
    }
    
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

2. Set permissions:
```bash
sudo gpasswd -a www-data ubuntu
```

3. Create symbolic link to enable site:
```bash
sudo ln -s /etc/nginx/sites-available/[project_name] /etc/nginx/sites-enabled
```

### Start Nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```

### Configure Port 80

1. Add ingress rule for port 80
2. Configure Ubuntu firewall:
```bash
sudo iptables -I INPUT 5 -p tcp --dport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
```

## 6. SSL Configuration (through cloudflare)

## Maintenance Commands

### Restart Services After Changes
```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo nginx -s reload
sudo systemctl restart nginx
```

### Check Service Status
```bash
sudo systemctl status gunicorn.socket
sudo systemctl status gunicorn
sudo systemctl status nginx
```

Remember to replace placeholders (indicated by `[brackets]`) with your actual values before using the commands.
