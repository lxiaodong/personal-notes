---
title: "关于 client_max_body_size client_body_buffer_size 配置"
tags: 
- nginx
- client_body_buffer_size
- client_max_body_size
- 随笔
weight: 10
description: "Nginx 关于client_max_body_size client_body_buffer_size配置"
---

## 1.问题
今天遇到一个问题，请求nginx提示如下问题？

    2022/09/06 17:56:24 [warn] 30797#0: *228364 a client request body is buffered to a temporary file /usr/local/nginx/client_body_temp/0000000134, client: 47.119.124.57, server: 47.106.69.23, request: "POST /api/news/publish HTTP/1.1", host: "47.106.69.239:8186"
    /usr/local/nginx/conf/vhost/lua/token.lua:158: in function </usr/local/nginx/conf/vhost/lua/token.lua:1>, client: 47.119.124.57, server: 47.106.69.23, request: "POST /api/news/publish HTTP/1.1", host: "47.106.69.239:8186"
    
## 2.原因
初步判断是否与client_body_buffer_size有关？然后了解了相关的知识（参考 [nginx官网](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) ），如下：

{{% notice info %}}
#### 2.1 client_max_body_size
```text
    句法：	client_max_body_size size;
    默认：	client_max_body_size 1m；
    语境：	http, server,location 

    设置客户端请求正文的最大允许大小。如果请求中的大小超过配置的值，则会向客户端返回 413（请求实体太大）错误。请注意，浏览器无法正确显示此错误。设置size为 0 将禁用对客户端请求正文大小的检查。
```
#### 2.2 client_body_buffer_size
```text
    句法：	client_body_buffer_size size;
    默认：	client_body_buffer_size 8k|16k；
    语境：	http, server,location

    设置读取客户端请求正文的缓冲区大小。如果请求正文大于缓冲区，则将整个正文或仅其部分写入 临时文件。默认情况下，缓冲区大小等于两个内存页。这是 x86、其他 32 位平台和 x86-64 上的 8K。在其他 64 位平台上通常为 16K。
```
{{% /notice %}}

## 3.解决方案调整client_max_body_size大小
```text   
    因为默认是16k, 将client_max_body_size 调整为50k, 就ok了。
```