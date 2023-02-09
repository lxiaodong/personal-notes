---
title: "通过redis验证是否需要反向代理"
tags: 
- lua
- redis
- proxy
- 随笔
weight: 10
description: "通过redis验证是否需要反向代理"
---

## 1.预想效果
{{% notice info %}}
#### 1.1 请求地址
http://localhost:8080/html/share/news?source=app_ios_3.5.8&news_id=112233

#### 1.2 反向后的地址
http://static.ijiwei.com/html/share/news/112233?source=app_ios_3.5

{{% /notice %}}

## 2.docker-compose配置
```Docker
version: "3.7"

networks:
  jiwei_network:
    driver: bridge

services:
  jiwei-fpm:
    build:
      context: .
      dockerfile: Dockerfile
    networks:
      - jiwei_network
    restart: always
    environment:
      - UNAME=work
      - UID=1000
      - FPM_PORT=9000
      - PM_TYPE=dynamic
      - PM_MAX_CHILDREN=45
      - PM_MIN_SPARE=15
      - PM_MAX_SPARE=40
      - PM_START-SERVERS=20
      - UPLOAD_MAX_FILESIZE=128M
      - POST_MAX_SIZE=128M
      - MEMORY_LIMIT=1G
      - FPM_ERROR=E_ALL & ~ E_STRICT & ~ E_NOTICE
    volumes:
      # 开发环境挂载目录，会覆盖掉容器内的vendor目录，需要去内部执行以下composer install
      - ./:/app/
      - ./log/:/app/log/
      - ./log/php-fpm-log:/log/
    container_name:
      jiwei-fpm

  nginx:
    image: openresty/openresty:alpine
    ports:
      - "8080:8081"
    networks:
      - jiwei_network
    depends_on:
      - jiwei-fpm
    volumes:
      - "./static/nginx:/var/log/nginx"
      - "./jiwei.conf:/etc/nginx/conf.d/jiwei.conf"
      - "./redirect.lua:/etc/nginx/conf.d/redirect.lua"

```

## 3. nginx配置
```nginx
server
{
    listen       8081;
    root /app;

    access_log /var/log/nginx/jiwei.access.log;
    error_log  /var/log/nginx/jiwei.error.log notice;

    index  index.php;

    location ~* .*\.(gif|jpg|jpeg|png|bmp|swf|ico|otf|ttf)$ {
        expires 30d;
        try_files $uri =404;
    }

    location ~* .*\.(js|css)?$ {
        expires 1d;
        try_files $uri =404;
    }

    # url匹配
    location ~^/html/share/news.*$ {
        default_type text/html;
        set $redirect_env "dev";
        set $redirect_type "shareNews";
        rewrite_by_lua_file  /etc/nginx/conf.d/news_hot.lua;
        proxy_pass http://static.ijiwei.com;
    }

    location @php {
            rewrite "^/(.*)" /public/index.php/$1 break;
            fastcgi_pass   jiwei-fpm:9000;
            fastcgi_index  index.php;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
            add_header Access-Control-Allow-Origin *;
    }

    # 重点注意这里
    location / {
         content_by_lua_block {
             ngx.exec("@php");
         }
    }

    charset utf-8;

    location = /favicon.ico { access_log off; log_not_found off; }
    location ~ /\.ht {
        deny all;
    }
}
```

## 4.lua脚本Redis验证方式
```lua
local redis = require("resty.redis")
local cjson = require("cjson")

local redirectClient = {
    env = "",
    type = "",
    arg = {},
    redis = {},
    const = {},
}

function redirectClient:new(obj, env, type)
    obj = obj or {}
    setmetatable(obj, self)
    self.__index = self
    self.env = env
    self.type = type
    self:init()
    return obj
end

function redirectClient:init()
    self:newConst()
    self:redirect()
    self:initArg()
    self:newRedis()
end

function redirectClient:newConst()
    local function Const(const_table)
        local mt = {
            __index = function(t, k)
                return const_table[k]
            end
        }
        return mt
    end
    local t = {}
    local const_params = {
        TYPE_SHARE_NEWS = "shareNews",
    }
    setmetatable(t, Const(const_params))
    self.const = t
end

function redirectClient:redirect()
    if self.type == self.const.TYPE_SHARE_NEWS then
        local arg = ngx.req.get_uri_args()
        local url = ngx.var.request_uri
        local m, err = ngx.re.match(url, "(news/[0-9]+)")
        if type(m) == "nil" then
            if arg.news_id ~= nil then
                local url = "/html/share/news/" .. arg.news_id
                local tmpUrlTab = {}
                for key, val in pairs(arg) do
                    if key ~= "news_id" then
                        local item = ""
                        if type(val) == "table" then
                             for k, v in pairs(val) do
                                item = key .. "=" .. v
                                table.insert(tmpUrlTab, item)
                             end
                        else
                             item = key .. "=" .. val
                             table.insert(tmpUrlTab,  item)
                        end
                    end
                end
                local tmpUrl = table.concat(tmpUrlTab, "&")
                if tmpUrl ~= "" then
                    url = url .. "?" .. tmpUrl
                end
                return ngx.redirect(url)
            end
        end
    end
end

function redirectClient:initArg()
    if self.type == self.const.TYPE_SHARE_NEWS then
        local url = ngx.var.request_uri
        -- m[0] = news_id=123 or news/123
        -- m[1] = news/ or news_id=
        -- m[2] = 123
        local m, err = ngx.re.match(url, "(news/|news_id=)([0-9]+)")
        local arg = ngx.req.get_uri_args()
        self.arg.news_id = m[2]
        self.arg.source = arg.source
    end
end

function redirectClient:newRedis()
    local config = self:getRedisConfig()
    if config == nil then
        ngx.log(ngx.ERR, "not fount config")
        return
    end

    local red = redis.new()
    -- 分别设置connect_timeout、send_timeout、read_timeout的超时阈值（ms）
    red:set_timeouts(1000, 1000, 1000)

    local ok, err = red:connect(config.host, config.port)
    if not ok then
        ngx.log(ngx.ERR, "failed to connect: " .. err)
        return
    end

    local res, err = red:auth(config.pwd)
    if not res then
        ngx.log(ngx.ERR, "failed to authenticate: " .. err)
        return
    end

    self.redis = red
    return
end

function redirectClient:getRedisConfig()
    local configs = {
        test = {host = "47.106.69.239", port = "6379", pwd = "j2drz5DQqZRYPCtT."},
        pro = {host = "172.18.172.160", port = "6379", pwd = "j2drz5DQqZRYPCtT"}
    }

    local config = nil
    for key, val in pairs(configs) do
        if key == self.env then
            config = val
            break
        end
    end
    return config
end

function redirectClient:keepalive()
    -- set_keepalive(max_idle_timeout, pool_size)
    self.redis:set_keepalive(60000, 1000)
end

function redirectClient:shareNews()
    if next(self.redis) == nil then
        ngx.log(ngx.NOTICE, "request php, news_id:", self.arg.news_id)
        return ngx.exec("@php")
    else
        local newId = self.arg.news_id
        local cacheKey = "NEWS:HOT:ID:" .. newId
        local resp, err = self.redis:get(cacheKey)

        -- 放入参数池
        self:keepalive()

        if not resp then
            ngx.log(ngx.NOTICE, "request php, news_id:", self.arg.news_id)
            return ngx.exec("@php")
        end

        local uri = ngx.var.uri
        if resp ~= ngx.null then
            ngx.log(ngx.NOTICE, "request static, news_id:", self.arg.news_id)
            uri = "/sibylla/".. self.env .."/n/" .. newId .. '.html'
            ngx.req.set_uri_args("") -- 设置无请求参数，否则七牛cdn会带参数缓存
            ngx.req.set_uri(uri);
        else
            ngx.log(ngx.NOTICE , "request php, news_id:", self.arg.news_id)
            return ngx.exec("@php")
        end
    end
end

function redirectClient:run()
    if self.type == self.const.TYPE_SHARE_NEWS then
        self:shareNews()
    else
        ngx.log(ngx.NOTICE, "request php")
        return ngx.exec("@php")
    end
end

local env = ngx.var.redirect_env
local type = ngx.var.redirect_type
local client = redirectClient:new(nil, env, type)
client:run()
```