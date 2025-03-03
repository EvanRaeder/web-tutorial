# Django Deployment Guide for Oracle Cloud Infrastructure (OCI)

This guide walks through deploying a Django application on Oracle Cloud Infrastructure using Ubuntu, Gunicorn, and Nginx.

```mermaid
graph TD;
    A[Client] --> B(Nginx);
    B --> C{gunicorn};
    C --> D[Django Application];
    E{Static} --> B;
    D --> C;
    C --> B;
    B --> A;
```

## Prerequisites

- An Oracle Cloud Infrastructure (OCI) account
- A Django application ready for deployment
- Ubuntu VM set up on OCI
- Your DNS

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
5. **Create Subnets**:
    ![Subnet Creation](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-3.png?raw=true?)
    ![Subnet Creation](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-4.png?raw=true?)
6. **Check CIDR**:
![CIDR](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-5.png?raw=true?)
7. **Security Lists**: Set up security lists to control the traffic allowed to and from the subnets.
    - **Create a New List**
    ![Create Security List](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-6.png?raw=true?)
    - **Add Rules**
    ![Create Rules](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-7.png?raw=true?)
    ![Create Rules](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-8.png?raw=true?)
8. **Internet Gateway**: Create an Internet Gateway to allow traffic to and from the internet.
![Create Gateway](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/vcn-9.png?raw=true?)

Your Virtual Cloud Network is now set up and ready for use.

## 1. Initial Server Setup

### Create Instance

![Create Instance](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/OCI-1.png?raw=true?)
Select Shape
![Select Shape](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/OCI-2.png?raw=true?)
Choose the created vnic and select Public IP info
![Select Vnic](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/OCI-3.png?raw=true?)
Save Private Key
![Save Key](https://github.com/EvanRaeder/web-tutorial/blob/main/docs/media/OCI-4.png?raw=true?)

### Add the Key (Client)

Keep the key somewhere safe (`User/.ssh`)

```bash
ssh-add [path_to_private_key]
```

_If ssh agent not running you have to run `Start-Service ssh-agent` (Win) or `eval "ssh-agent"`_

### Connect to Server

```bash
ssh ubuntu@[ip_address]
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

## 3. Make sure IP's are added to DNS Provider

Log in to your domain registrar's control panel: This is where you manage your domain settings. Add the following records.

_Depending on DNS provider name may be blank instead of @_

| Type  | Name | Value             |
|-------|------|-------------------|
| A     | @    | Enter the IPv4 address |
| AAAA  | @    | Enter the IPv6 address |

_If you want www routing as well follow the same steps with name:www_


## 4. Firewall Configuration

### Configure Ubuntu Firewall

```bash
sudo apt-get install iptables-persistent
sudo iptables -I INPUT 5 -p tcp --dport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
sudo iptables -I INPUT 5 -p tcp --dport 443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
```

**Note:** If you are going to dissallow http then **do not** allow port 80

_However; It may be useful for testing._

## 5. Gunicorn Setup

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

## 6. Nginx Setup

### Install Nginx

```bash
sudo apt-get install nginx
```

### Configure Django Settings For Deployment

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
    server_name [your_domain] [www.yourdomain];

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

_You may need to make the static folder readable by www-data as well_

3. Create symbolic link to enable site:

```bash
sudo ln -s /etc/nginx/sites-available/[project_name] /etc/nginx/sites-enabled
```

### Start Nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```

## 7. SSL Configuration (through certbot)

It is recommended that you follow the seperate guide for setting up cloudflare.
If you would not like to use cloud flare you can use certbot instead.

### Install certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

### Get SSL certbot

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Follow the setup instructions when it asks **Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.** choose **2: Redirect**

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

