---
title: "Openresty_install"
date: 2022-01-04T17:09:47+08:00
draft: true
---

# install openresty
[install-openresty](https://openresty.org/en/linux-packages.html)

# install auto ssl
```shell
apt install luarocks make
luarocks install lua-resty-auto-ssl

# create Self-signed certificate
mkdir /usr/local/openresty/ssl/
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
  -subj '/CN=sni-support-required-for-valid-ssl' \
  -keyout /usr/local/openresty/ssl/resty-auto-ssl-fallback.key \
  -out /usr/local/openresty/ssl/resty-auto-ssl-fallback.crt

# create ssl directory
mkdir -p /usr/local/openresty/ssl/resty-auto-ssl
chown nobody:nogroup /usr/local/openresty/ssl/resty-auto-ssl
```

# configure
vim /usr/local/openresty/nginx/conf/nginx.conf
```
user      nobody nogroup;
worker_processes    auto;
worker_rlimit_nofile    102400;
timer_resolution        1000ms;
pid           logs/nginx.pid;

events {
    worker_connections  102400;
    multi_accept on;
    use epoll;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    types_hash_max_size 2048;
    server_tokens off;
    keepalive_timeout  65;
    include       mime.types;
    more_clear_input_headers "X-Ops-Role";
    default_type  application/octet-stream;
    max_ranges 1;

    server {
        listen 80;
        listen 443 ssl;
        server_name localhost;
        ssl_certificate /usr/local/openresty/ssl/resty-auto-ssl-fallback.crt;
        ssl_certificate_key /usr/local/openresty/ssl/resty-auto-ssl-fallback.key;
        more_set_headers 'Server: xmqos';
        location / { add_header Content-Type text/plain; return 200 'ok.';}
        location /nginxstatus { stub_status; access_log off; allow 127.0.0.1; deny all;}
    }

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $bytes_sent $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_transfer_encoding" "$http_x_forwarded_for" '
                      '"$upstream_addr" $host $sent_http_x_reqid "$upstream_response_time" $request_time $request_length '
                      '$http_x_stat $http_x_estat $server_addr';

    map $time_iso8601 $loghour {
     '~^(?<ymdh>\d{4}-\d{2}-\d{2}T\d{2})' $ymdh;
     default 'iso8601_not_match';
    }

    error_log   logs/error.log;
    access_log  logs/access.log  main;

    limit_conn_zone $http_host zone=servicelimit:10m;
    limit_conn_zone $http_host zone=limitspeed:50m;
    limit_conn_log_level error;

    # Gzip Settings
    gzip_min_length 1000;
    gzip_comp_level 8;
    gzip_proxied any;
    gzip_types text/plain text/css text/javascript text/xml application/x-javascript application/json application/xml application/xml+rss application/javascript;
    gzip  off;
    server_names_hash_max_size 5120;

    # ssl Settings
    ssl_ciphers         HIGH:!aNULL:!MD5:!DES;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_session_cache   shared:SSL:10m;
    lua_shared_dict auto_ssl 1m;
    lua_shared_dict auto_ssl_settings 64k;
    resolver 8.8.8.8;
    init_by_lua_block {
  auto_ssl = (require "resty.auto-ssl").new()
  auto_ssl:set("dir", "/usr/local/openresty/ssl/resty-auto-ssl")
  auto_ssl:set("hook_server_port", 8999)
  auto_ssl:set("renew_check_interval", 172800)
  auto_ssl:set("allow_domain", function(domain)
          return ngx.re.match(domain, "(cloudrushpool.com)$", "ijo")
      end)
      auto_ssl:init()
    }

    init_worker_by_lua_block {
  auto_ssl:init_worker()
    }
    server {
        listen 127.0.0.1:8999;
        client_body_buffer_size 128k;
        client_max_body_size 128k;
        location / {
            content_by_lua_block {
                auto_ssl:hook_server()
            }
        }
    }

    include /usr/local/openresty/nginx/conf.d/*.conf;
}
```

vim /usr/local/openresty/nginx/conf.d/sample.conf
```
upstream sample-service {
    server localhost:8888 max_fails=3;
}
server {
    listen 80;
    server_name sample-service.aaa.com;
    access_log  logs/sample-service.aaa.com.log main;

    client_max_body_size 1024m;
    more_set_headers 'Server: xmqos';
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_hide_header "X-Tag";
    set $real_remote_addr $remote_addr;
    proxy_set_header X-Real-IP $real_remote_addr;

    location /.well-known/acme-challenge/ {
      content_by_lua_block {
        auto_ssl:challenge_server()
      }
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name sample-service.aaa.com;
    access_log  logs/sample-service.aaa.com.log main;

    ssl_certificate_by_lua_block {
        auto_ssl:ssl_certificate()
    }
    ssl_certificate /usr/local/openresty/ssl/resty-auto-ssl-fallback.crt;
    ssl_certificate_key /usr/local/openresty/ssl/resty-auto-ssl-fallback.key;

    client_max_body_size 1024m;
    more_set_headers 'Server: xmqos';
    proxy_send_timeout 500s;
    proxy_read_timeout 500s;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_hide_header "X-Tag";
    set $real_remote_addr $remote_addr;
    proxy_set_header X-Real-IP $real_remote_addr;

    add_header Strict-Transport-Security max-age=63072000;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    gzip on;
    gzip_http_version 1.1;
    gzip_types text/plain text/css text/javascript application/javascript application/x-javascript image/jpeg image/gif image/png;
    gzip_min_length 1024;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header Referer $http_referer;
        proxy_pass http://sample-service;
    }
}
```
