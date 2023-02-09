---
title: "lua模块引用resty.redis"
tags: 
- nginx
- lua
- resty.redis
- 随笔
weight: 3
description: "Nginx lua模块引用resty.redis扩展"
---

## 1. 问题
lua脚本需要使用resty.redis扩展包，需要安装一下。

## 2. 安装

#### 2.1下载lua-resty-redis
```bash
cd /data/soft/nginx # 如果不存在可 mkdir
wget https://github.com/openresty/lua-resty-redis/archive/refs/heads/master.zip
mv master.zip lua-resty-redis.zip
unzip lua-resty-redis.zip
```

#### 2.2 查看redis.lua存放目录
```lua
local a = require("resty.redis")

--输出如下：
stdin:1: module 'resty.redis' not found:
        no field package.preload['resty.redis']
        no file './resty/redis.lua'
        no file '/usr/share/lua/5.1/resty/redis.lua'
        no file '/usr/local/luajit/share/luajit-2.0.4/resty/redis.lua'
        no file '/usr/share/lua/5.1/resty/redis/init.lua'
        no file '/usr/lib64/lua/5.1/resty/redis.lua'
        no file '/usr/lib64/lua/5.1/resty/redis/init.lua'
        no file './resty/redis.so'
        no file '/usr/lib64/lua/5.1/resty/redis.so'
        no file '/usr/lib64/lua/5.1/loadall.so'
        no file './resty.so'
        no file '/usr/lib64/lua/5.1/resty.so'
        no file '/usr/lib64/lua/5.1/loadall.so'
stack traceback:
        [C]: in function 'require'
        stdin:1: in main chunk
        [C]: ?
```


#### 2.3 将redis.lua 放入lua自动寻找的文件目录中
```bash
cd lua-resty-redis-master/lib/resty/
cp ./redis.lua /usr/local/luajit/share/luajit-2.0.4/resty/
```