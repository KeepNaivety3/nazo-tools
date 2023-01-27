# Install NexusPHP on CentOS Stream

## 1. Preparation for installing NexusPHP

From the documentation, twe have some installation environments that need to be prepared. 

> - PHP: `8.0`, must have extensions: `bcmath, ctype, curl, fileinfo, json, mbstring, openssl, pdo_mysql, tokenizer, xml, mysqli, gd, redis, pcntl, sockets, posix, gmp, opcache`
> - Mysql: `5.7` latest version or above is recommended
> - Redis: `2.6.12` or above

### 1.1 Install MySQL

```bash
# Install MySQL
yum install -y https://dev.mysql.com/get/mysql80-community-release-el8-4.noarch.rpm
yum install -y mysql-server

# Enable and start mysql service
systemclt enable --now mysqld.service

# Setup for mysql
# just set a password and choose the default option.
mysql_secure_installation
```

### 1.2 Install Redis

```bash
# Install redis
yum install -y redis

# Enable and start redis service
systemctl enable --now redis.service
```

### 1.3 Install PHP

The PHP version of yum's default installation source is lower than 8.0. For a better user experience in the future, the PHP source installation of `Remi's RPM repository` is selected here.

The reason why the yum repo source software version is relatively backward may be because the default module is a low version. If I have time in the future, I will write a separate post to introduce the yum module.

> Besides individual RPM packages, the AppStream repository contains modules. A module is a set of RPM packages that represent a component and are usually installed together. A typical module contains packages with an application, packages with the application-specific dependency libraries, packages with documentation for the application, and packages with helper utilities.

```bash
# Install Remi's RPM repository
yum install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm

# List of the PHP module
yum module list php
CentOS Stream 8 - AppStream
Name               Stream                    Profiles                                Summary                            
php                7.2 [d]                   common [d], devel, minimal              PHP scripting language             
php                7.3                       common [d], devel, minimal              PHP scripting language             
php                7.4                       common [d], devel, minimal              PHP scripting language             
php                8.0                       common [d], devel, minimal              PHP scripting language             

Remi's Modular repository for Enterprise Linux 8 - x86_64
Name               Stream                    Profiles                                Summary                            
php                remi-7.2                  common [d], devel, minimal              PHP scripting language             
php                remi-7.3                  common [d], devel, minimal              PHP scripting language             
php                remi-7.4                  common [d], devel, minimal              PHP scripting language             
php                remi-8.0 [e]              common [d], devel, minimal              PHP scripting language             
php                remi-8.1                  common [d], devel, minimal              PHP scripting language             
php                remi-8.2                  common [d], devel, minimal              PHP scripting language             

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled

# Change the default module to remi-8.0
yum module enable php:remi-8.2

# Install PHP8.0
yum install -y php

# Install PHP extensions
yum install -y php-bcmath php-common php-mbstring php-pdo php-mysqlnd php-xml php-gd php-pecl-redis5 php-cli php-process php-gmp php-opcache
```

## 2. Setup for NexusPHP

### 2.1 Clone NexusPHP from Github

Clone xiaomlove/nexusphp and switch to the latest release tab before installing it, or just download the latest release. Be sure to switch to a release for installation when cloning. Do not use the latest development code!

```bash
git clone -b 1.7 https://github.com/xiaomlove/nexusphp.git
```

### 2.2 Create a database

First login to Mysql, create a new database, and choose utf8 + utf8_general_ci or utf8mb4 + utf8mb4_general_ci for the charset and sorting rules (collate). The latter supports storing emoji expressions, the former does not.

```bash
mysql -u root -p
```

```mysql
create database `nexusphp` default charset=utf8mb4 collate utf8mb4_general_ci;
```

### 2.3 Install Nginx

```bash
# Install Nginx
yum install -y nginx

# Enable and start Nginx
systemctl enable --now nginx.service
```

### 2.4 Configure the web server

To enable https, you first have to prepare the certificate.

```nginx
server {
    listen 443 ssl;
    ssl_certificate /SOME/PATH/DOMAIN.pem;
    ssl_certificate_key /SOME/PATH/DOMAIN.key;

    # whichever is true
    root /RUN_PATH; 

    server_name DOMAIN;

    location / {
        index index.html index.php;
        try_files $uri $uri/ /nexus.php$is_args$args;
    }

    # Filament
    location ^~ /filament {
        try_files $uri $uri/ /nexus.php$is_args$args;
    }

    location ~ \.php {
        # whichever is true
        fastcgi_pass unix:/run/php-fpm/www.sock; 
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    access_log /var/log/nginx/DOMAIN.access.log;
    error_log /var/log/nginx/DOMAIN.error.log;
}
# http jump https
server {
    if ($host = DOMAIN) {
        return 301 https://$host$request_uri;
    }
    server_name DOMAIN;
    listen 80;
    return 404;
}
```

