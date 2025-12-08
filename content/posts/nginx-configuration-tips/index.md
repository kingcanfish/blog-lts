---
title: nginx配置的一些坑
date: 2019-07-05 09:08:00
tag: nginx
description: nginx配置的一些坑
---

昨天给域名上https时候nginx配置遇到了一些坑  
~~其实是我自己垃圾~~  
有时间再更  
---
***
# 2019/07/06更新  
***
 ## 1.首先我碰到了  
>### nginx -s reload  
之后报open xxxx/xxx/nginx.pid faild的问题
>### nginx -c /etc/nginx/nginx.conf
解决

## 2.给域名上ssl证书
在nginx.conf配置文件里加入如下代码  
```
 server {
         listen       443;
         server_name  example.com;
         ssl on;

         ssl_certificate      yourssl.pem;
         ssl_certificate_key  yourssl.key;

         # ssl_session_cache    shared:SSL:1m;
         ssl_session_timeout  5m;
         ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;

         ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
         #ssl_ciphers  HIGH:!aNULL:!MD5;
         ssl_prefer_server_ciphers  on;

         location / {
                 root   /home/git/projects/blog;
                 index  index.html index.htm;
   }
}


```
### 因为https协议默认监听的是443端口所以位置文件需要监听443端口
### 然后再将访问http协议跳转到https协议上
### 继续加入如下代码：
~~~
    server {
    listen 80;
    server_name   example.com;
    return      301 https://$server_name$request_uri;
}
~~~

## 3.注意配置文件的位置

**不同的nginx版本配置文件的路径可能不同，我的配置文件就存放在/etc/nginx/nginx.conf**
**而且要注意nginx.conf中有没有include其他的配置文件，比如**  
> include  conf.d/*.conf

**就包含了该文件夹内所有的配置文件，要注意**