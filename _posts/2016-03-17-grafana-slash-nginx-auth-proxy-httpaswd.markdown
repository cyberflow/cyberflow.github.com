---
layout: post
title: "Grafana/nginx auth proxy httpaswd"
date: 2016-03-17 17:25
comments: true
tags: [monitoring, linux, grafana, nginx, auth]
categories: [linux, monitoring]
published: true
---

Краткое описание настройки [Grafana](http://grafana.org) (> 2.0) и nginx с использованием auth basic авторизации через файлы htpasswd.

Необходимые настройки конфига Grafana:

```
[server]
protocol = http
http_port = 3000
domain = localhost
http_addr = 127.0.0.1
...
[auth.basic]
enabled=false
[users]
allow_sign_up = false
auto_assign_org = true
auto_assign_org_role = Editor
[auth.proxy]
enabled = true
header_name = X-WEBAUTH-USER
auto_sign_up = true
```
<!--more-->
Настройка nginx:

```
server {
  listen                <server_ip>:80;

  server_name           <server_hostname>;
  access_log            /var/log/nginx/grafana.access.log;

  location / {
    auth_basic            'Restricted';
    auth_basic_user_file  <path/to/htpasswd_file>;
    proxy_set_header X-WEBAUTH-USER $remote_user;
    proxy_pass http://grafana;
  }
}
```

После сделать рестарт сервера Grafana и nginx.
