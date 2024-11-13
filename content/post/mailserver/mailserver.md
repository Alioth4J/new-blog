---
title: An attempt to set up a mailserver, but ultimately failed
description: Just record
date: 2024-11-13
slug: mailserver
image: 
categories:
    - Linux
---

## My machine
Alibaba Cloud: centos_7_9_x64_20G_alibase_20240628.vhd  
The following commands are executed with root privileges.  

## Install docker-ce and docker-compose

> Use a mirror if the default url is not accessible.

Set up the docker repository:
```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install docker and docker-compose with yum:
```bash
yum install docker-ce docker-compose
```

Start and enable docker:
```bash
systemctl start docker
systemctl enable docker
```

Then check the status of docker:
```bash
systemctl status docker
```

## Create the project directory
```bash
mkdir mailserver
cd mailserver
```

## Prepare sql script
I use MySQL to store the data of email accounts.  

mysql-init/init.sql:

```sql
create database if not exists mailserver;

use mailserver;

create table if not exists user (
  id int auto_increment primary key,
  email varchar(255) not null unique,
  password varchar(255) not null,
  active boolean default true
) engine=innodb;

insert into user (email, password) values ('test@yourdomain.com', 'password');
```

## Create docker-compose.yml
Use MySQL, Postfix and Dovecot.  
```yml
version: '3'

services:
  mysql:
    image: mysql:5.7
    container_name: mail_mysql
    environment:
      MYSQL_ROOT_PASSWORD: examplepassword
      MYSQL_DATABASE: mailserver
      MYSQL_USER: root
      MYSQL_PASSWORD: examplepassword
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    networks:
      - mailnetwork

  postfix:
    image: tvial/docker-mailserver:latest
    container_name: mail_postfix
    environment:
      MAIL_DOMAIN: mydomain.com
      MAIL_HOST: mail.mydomain.com
      MYSQL_HOST: mysql
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: examplepassword
      MYSQL_DB: mailserver
      MYSQL_USER_TABLE: user
      MYSQL_USER_FIELD: email
      MYSQL_PASSWORD_FIELD: examplepassword
    depends_on:
      - mysql
    ports:
#      - "25:25"     # SMTP
      - "26:25"     # use another port as the isp bans port 25
      - "587:587"   # Submission
      - "993:993"   # IMAPS
    volumes:
      - ./data/postfix:/var/mail
    networks:
      - mailnetwork

  dovecot:
    image: dovecot/dovecot
    container_name: mail_dovecot
    environment:
      MYSQL_HOST: mysql
      MYSQL_PORT: 3306
      MYSQL_USER: root
      MYSQL_PASSWORD: examplepassword
      MYSQL_DB: mailserver
      MYSQL_USER_TABLE: user
      MYSQL_USER_FIELD: email
      MYSQL_PASSWORD_FIELD: examplepassword
    depends_on:
      - mysql
    volumes:
      - ./data/dovecot:/var/mail
    networks:
      - mailnetwork

networks:
  mailnetwork:
    driver: bridge

volumes:
  mysql-data:
```

## Start docker-compose
```bash
docker-compose up -d
```

## Configure DNS and open required port
DNS:
- A: `mydomain.com` -> server ip address
- A: `mail.mydomain.com` -> server ip address
- MX: `mydomain.com` -> `mail.mydomain.com`

// Setting up firewall ports omitted

## Test
Something went wrong:
```bash
$ telnet mail.mydomain.com 26
Trying ${ip}...
Connected to mail.mydomain.com.
Escape character is '^]'.
Connection closed by foreign host.
```

[Similar problem found on StackOverflow with no solutions](https://stackoverflow.com/questions/78777768/telnet-connection-to-smtp-server-immediately-closed-but-nc-connection-succeeds)

After several struggles, I'm still unable to solve the problem.  

## Conclusion
Maybe I will solve the problem later.  

**Whatever, embrace the lessons from the journey and happy coding!**  
