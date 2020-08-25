---
title: "Deploying Django Project"
date: 2020-07-15T17:33:22+05:30
draft: false
author: "sreevardhan"
# cover: "https://images.unsplash.com/photo-1571786256017-aee7a0c009b6?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=800&q=80"
tags:
  - python
  - django
  - nginx
  - gunicorn
categories:
  - python
  - deployment
keywords: "python gunicorn supervisor systemd nginx servers"
description: "Django Deployment guide using Gunicorn and Nginx on Digital Ocean."
slug: "deploying-django-project"
showFullContent: ""
---

<!-- # Django Deployment guide using Gunicorn and Nginx. -->

This is a summarized document from this [digital ocean doc](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-18-04)

Any commands with "\$" at the beginning run on your local machine and any "#" run when logged into the server

# Security & Access

### Creating SSH keys (Optional)

You can choose to create SSH keys to login if you want. If not, you will get the password sent to your email to login via SSH

To generate a key on your local machine

```console
$ ssh-keygen
```

Hit enter all the way through and it will create a public and private key at

```console
~/.ssh/id_rsa
~/.ssh/id_rsa.pub
```

You want to copy the public key (.pub file)

```console
$ cat ~/.ssh/id_rsa.pub
```

Copy the entire output and add as an SSH key for Digital Ocean

### Login To Your Server

If you setup SSH keys correctly the command below will let you right in. If you did not use SSH keys, it will ask for a password. This is the one that was mailed to you

```console
$ ssh root@YOUR_SERVER_IP
```

### Create a new user

It will ask for a password, use something secure. You can just hit enter through all the fields. I used the user "djangoadmin" but you can use anything

```console
# adduser djangoadmin
```

### Give root privileges

```console
# usermod -aG sudo djangoadmin
```

### SSH keys for the new user

Now we need to setup SSH keys for the new user. You will need to get them from your local machine

### Exit the server

You need to copy the key from your local machine so either exit or open a new terminal

```console
# exit
```

You can generate a different key if you want but we will use the same one so lets output it, select it and copy it

```console
$ cat ~/.ssh/id_rsa.pub
```

### Log back into the server

```console
$ ssh root@YOUR_SERVER_IP
```

### Add SSH key for new user

Navigate to the new users home folder and create a file at '.ssh/authorized_keys' and paste in the key

```console
# cd /home/djangoadmin
# mkdir .ssh
# cd .ssh
# nano authorized_keys
Paste the key and hit "ctrl-x", hit "y" to save and "enter" to exit
```

### Login as new user

You should now get let in as the new user

```console
$ ssh djangoadmin@YOUR_SERVER_IP
```

### Disable root login

```console
# sudo nano /etc/ssh/sshd_config
```

### Change the following

```console
PermitRootLogin no
PasswordAuthentication no
```

### Reload sshd service

```console
# sudo systemctl reload sshd
```

# Simple Firewall Setup

See which apps are registered with the firewall

```console
# sudo ufw app list
```

Allow OpenSSH

```console
# sudo ufw allow OpenSSH
```

### Enable firewall

```console
# sudo ufw enable
```

### To check status

```console
# sudo ufw status
```

We are now done with access and security and will move on to installing software

# Software

## Update packages

```console
# sudo apt update
# sudo apt upgrade
```

## Install Python 3, Postgres & NGINX

```console
# sudo apt install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```

# Postgres Database & User Setup

```console
# sudo -u postgres psql
```

You should now be logged into the pg shell

### Create a database

```sql
CREATE DATABASE db_prod;
```

### Create user

```sql
CREATE USER dbadmin WITH PASSWORD 'test1234';
```

### Set default encoding, tansaction isolation scheme (Recommended from Django)

```sql
ALTER ROLE dbadmin SET client_encoding TO 'utf8';
ALTER ROLE dbadmin SET default_transaction_isolation TO 'read committed';
ALTER ROLE dbadmin SET timezone TO 'UTC';
```

### Give User access to database

```sql
GRANT ALL PRIVILEGES ON DATABASE db_prod TO dbadmin;
```

### Quit out of Postgres

```sql
\q
```

# Vitrual Environment

You need to install the python3-venv package

```console
# sudo apt install python3-venv
```

### Create project directory

```console
# mkdir pyapps
# cd pyapps
```

### Create venv

```console
# python3 -m venv ./venv
```

### Activate the environment

```console
# source venv/bin/activate
```

# Git & Upload

### Pip dependencies

From your local machine, create a requirements.txt with your app dependencies. Make sure you push this to your repo

```console
$ pip freeze > requirements.txt
```

Create a new repo and push to it (you guys know how to do that)

### Clone the project into the app folder on your server (Either HTTPS or setup SSH keys)

```console
# git clone https://github.com/yourgithubname/your_project.git
```

## Install pip modules from requirements

You could manually install each one as well

```console
# pip install -r requirements.txt
```

# Local Settings Setup

Add code to your settings.py file and push to server

```python
try:
    from .prod_settings import *
except ImportError:
    pass
```

Create a file called **prod_settings.py** on your server along side of settings.py and add the following

- SECRET_KEY
- ALLOWED_HOSTS
- DATABASES
- DEBUG
- EMAIL

## Run Migrations

```console
# python manage.py makemigrations
# python manage.py migrate
```

## Create super user

```console
# python manage.py createsuperuser
```

## Create static files

```console
python manage.py collectstatic
```

### Create exception for port 8000

```console
# sudo ufw allow 8000
```

## Run Server

```console
# python manage.py runserver 0.0.0.0:8000
```

### Test the site at YOUR_SERVER_IP:8000

Add some data in the admin area

# Gunicorn Setup

Install gunicorn

```console
# pip install gunicorn
```

Add to requirements.txt

```console
# pip freeze > requirements.txt
```

### Test Gunicorn serve

```console
# gunicorn --bind 0.0.0.0:8000 your_project.wsgi
```

Your images, etc will be gone

### Stop server & deactivate virtual env

```console
ctrl-c
# deactivate
```

### Open gunicorn.socket file

```console
# sudo nano /etc/systemd/system/gunicorn.socket
```

### Copy this code, paste it in and save

```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### Open gunicorn.service file

```console
# sudo nano /etc/systemd/system/gunicorn.service
```

### Copy this code, paste it in and save

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=djangoadmin
Group=www-data
WorkingDirectory=/home/djangoadmin/apps/your_project
ExecStart=/home/djangoadmin/apps/venv/bin/gunicorn \
          --access-logfile - \
          --workers 2 \
          --bind unix:/run/gunicorn.sock \
          your_project.wsgi:application

[Install]
WantedBy=multi-user.target
```

### Start and enable Gunicorn socket

```console
# sudo systemctl start gunicorn.socket
# sudo systemctl enable gunicorn.socket
```

### Check status of guinicorn

```console
# sudo systemctl status gunicorn.socket
```

### Check the existence of gunicorn.sock

```console
# file /run/gunicorn.sock
```

# NGINX Setup

### Create project folder

```console
# sudo nano /etc/nginx/sites-available/your_project
```

### Copy this code and paste into the file

```nginx
server {
    listen 80;
    server_name YOUR_IP_ADDRESS;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/djangoadmin/pyapps/your_project;
    }

    location /media/ {
        root /home/djangoadmin/pyapps/your_project;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

### Enable the file by linking to the sites-enabled dir

```console
# sudo ln -s /etc/nginx/sites-available/your_project /etc/nginx/sites-enabled
```

### Test NGINX config

```console
# sudo nginx -t
```

### Restart NGINX

```console
# sudo systemctl restart nginx
```

### Remove port 8000 from firewall and open up our firewall to allow normal traffic on port 80

```console
# sudo ufw delete allow 8000
# sudo ufw allow 'Nginx Full'
```

### You will probably need to up the max upload size to be able to create listings with images

Open up the nginx conf file

```console
# sudo nano /etc/nginx/nginx.conf
```

### Add this to the http{} area

```nginx
client_max_body_size 20M;
```

### Reload NGINX

```console
# sudo systemctl restart nginx
```

# Domain Setup

Go to your domain registrar and create the following a record

```
@  A Record  YOUR_IP_ADDRESS
www  CNAME  example.com
```

### Go to local_settings.py on the server and change "ALLOWED_HOSTS" to include the domain

```python
ALLOWED_HOSTS = ['IP_ADDRESS', 'example.com', 'www.example.com']
```

### Edit /etc/nginx/sites-available/your_project

```nginx
server {
    listen: 80;
    server_name xxx.xxx.xxx.xxx example.com www.example.com;
}
```

### Reload NGINX & Gunicorn

```console
# sudo systemctl restart nginx
# sudo systemctl restart gunicorn
```

# Alternate : Deploy with supervisor

Install supervisor

```console
apt-get install supervisor
```

After installing supervisor create a supervisor conf file

```console
sudo vim /etc/supervisor/conf.d/django_project.conf
```

A sample conf file for django project

```
[program:project]
command=/path_to/venv/bin/gunicorn --workers 2 --bind unix:/path_where_you_want_the_sock_file_to_reside/project.sock project.wsgi
directory=/root/path_to_project/project
autostart=true
autorestart=true
stderr_logfile=/var/log/project.err.log
stdout_logfile=/var/log/project.out.log

[supervisord]
environment=
        DATABASE_URL="postgres://postgres:root@localhost:5432/db_name",
        DEBUG_STATUS="False",
```

The conf file specifies how the project should be run by the gunicorn and where the log files should be and .env files for running the project

After adding the conf file run the below commands

```console
sudo supervisorctl reread
sudo supervisorctl update
```

To check if the supervisor services are running use the below commmands

```console

sudo service supervisor status
sudo service supervisor restart
sudo service supervisor start

```

Nginx config will remain same

common nginx gotchas

For debugging statifiles check access and error logs for nginx at /etc/nginx/

Sometimes for staticfiles use alias instead of root with trailing slash

Difference between root and alias inside static block

https://stackoverflow.com/questions/10631933/nginx-static-file-serving-confusion-with-root-alias#:~:text=This%20difference%20exists%20in%20the,is%20appended%20to%20the%20alias.

based on static file settings your nginx conf could be

```nginx
server {
    listen 80;
    server_name YOUR_IP_ADDRESS;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        alias /your_project/staticfiles/;
    }

    location /media/ {
        root /pyapps/your_project;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/path_where_the_sock_file_resides/project.sock;
    }
}
```
