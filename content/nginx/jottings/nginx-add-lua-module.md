---
title: "添加lua模块"
tags: 
- nginx
- lua
- 随笔
weight: 1
description: "Nginx 添加lua模块, 具体可参考lua相关"
---
## 1. 问题
鉴权的一个需求，因为服务过多，重复的鉴权逻辑比较不友好，又暂无网关，便想到了可以在nginx中使用lua脚本做鉴权处理，后续如果使用网关，也可以增加到网关中，一劳永逸。

## 2. 方案
nginx中有很多的配置，所以只能在原有的基础上增加扩展

## 3. 安装扩展

#### 3.1下载安装LuaJIT-2.0.4.tar.gz，需配置环境变量
{{% notice note %}}
lua是一种解释语言，通过luajit可以即时编译lua代码到机器代码，得到很好的性能；
{{% /notice %}}

```bash
    cd /data/soft/nginx # 如果不存在可 mkdir

    wget -c http://luajit.org/download/LuaJIT-2.0.4.tar.gz

    tar xzvf LuaJIT-2.0.4.tar.gz

    cd LuaJIT-2.0.4

    make install PREFIX=/usr/local/luajit
    
    #添加环境变量!

    export LUAJIT_LIB=/usr/local/luajit/lib

    export LUAJIT_INC=/usr/local/luajit/include/luajit-2.0
```

#### 3.2 下载解压[ngx_devel_kit](https://github.com/vision5/ngx_devel_kit)，不需要安装
```bash
    cd /data/soft/nginx

    wget https://github.com/simpl/ngx_devel_kit/archive/v0.3.0.tar.gz

    tar -xzvf v0.3.0.tar.gz
```

#### 3.3 下载解压[lua-nginx-module](https://github.com/openresty/lua-nginx-module)，不需要安装
```bash
    cd /data/soft/nginx

    wget https://github.com/openresty/lua-nginx-module/archive/v0.10.9rc7.tar.gz

    tar -xzvf v0.10.9rc7.tar.gz
```

#### 3.4 下载nginx，编译安装
{{% notice tip %}}
[nginx下载地址：](http://nginx.org/download/)
注意: nginx 如果升级的话最好与你当前环境的的版本一直，查看nginx版本（nginx -v）
{{% /notice %}}

```bash
    cd /data/soft/nginx

    wget http://nginx.org/download/nginx-1.12.2.tar.gz # 因为我的环境是1.12.2 所以下载了这个版本，如果其他版本，可以直接更改版本号

    tar -xzvf nginx-1.12.2.tar.gz
```

#### 3.5 编译安装
{{% notice tip %}}

查看当前nginx的扩展（nginx -V）

```text
    nginx version: nginx/1.12.2
    built by gcc 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) 
    built with OpenSSL 1.0.2l  25 May 2017
    TLS SNI support enabled
    configure arguments: --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l
```

###### 3.5.1 只增加lua模块
如果是在原有nginx上面加lua模块的话，记得只执行make就可以了。

```bash
    cd /data/soft/nginx/nginx-1.12.2

    ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l --add-module=/data/soft/nginx/lua-nginx-module-0.10.9rc7 --add-module=/data/soft/nginx/ngx_devel_kit-0.3.0 --with-stream

    make
```

###### 3.5.2 覆盖原有的nginx
```bash
    cd /data/soft/nginx/nginx-1.12.2

    ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l --add-module=/data/soft/nginx/lua-nginx-module-0.10.9rc7 --add-module=/data/soft/nginx/ngx_devel_kit-0.3.0 --with-stream

    make && make install
```
{{% /notice %}}

#### 3.6 验证是否增加lua模块
```bash
    cd /data/soft/nginx/nginx-1.12.2/objs

    ./nginx -V
```
```nginx
    {
        nginx version: nginx/1.12.2
        built by gcc 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) 
        built with OpenSSL 1.0.2l  25 May 2017
        TLS SNI support enabled
        configure arguments: --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l --add-module=/data/soft/nginx/lua-nginx-module-0.10.9rc7 --add-module=/data/soft/nginx/ngx_devel_kit-0.3.0 --with-stream
    }
```

#### 3.7 替换nginx的执行文件&平滑升级
```bash
    cd /usr/local/nginx/sbin/nginx

    mv nginx nginxbk

    cp /data/soft/nginx/nginx-1.12.2/objs/nginx ./

    #平滑升级
    make upgrade
```

#### 3.8 验证nginx增加lua模块是否成功
```bash
    nginx -V #查看是否增加了lua相关模块

    nginx version: nginx/1.12.2
    built by gcc 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) 
    built with OpenSSL 1.0.2l  25 May 2017
    TLS SNI support enabled
    configure arguments: --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_v2_module --with-http_gzip_static_module --with-ipv6 --with-http_sub_module --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l --add-module=/data/soft/nginx/lua-nginx-module-0.10.9rc7 --add-module=/data/soft/nginx/ngx_devel_kit-0.3.0 --with-stream
```

#### 3.9 验证lua是使用成功
```bash
    vim /usr/local/nginx/conf/nginx.conf

    ###在server模块里面，随便找个地方添加下面的代码
    location /lua {
        default_type 'text/plain';

        content_by_lua 'ngx.say("hello, lua")';
    }
```

## 4.可能遇到的问题？
{{% notice info %}}
上述安装过程是已经解决了所有问题的方法，以下是可能遇到的问题？

#### 4.1 error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
```text
    方法1：
    执行下面的代码（注意前面的路径是你自己安装的luajit路径，正常的安装可能不一样，需要自己排查以下）

    ln -s /usr/local/luajit/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2

    方法2：
    执行下面的代码（同样还是注意前面的路径）

    echo "/usr/local/luajit/lib" > /etc/ld.so.conf.d/usr_local_lib.conf
```
#### 4.2 如果nginx原本有openssl，但是编辑的时候找不到，需要自己重新指定一下
```text
    cd /data/soft/nginx

    wget http://soft.vpser.net/lnmp/lnmp1.4-full.tar.gz #下载安装包

    tar zxvf lnmp1.4-full.tar.gz

    cd lnmp1.4-full/src

    tar zxvf  openssl-1.0.2l.tar.gz

    ...
    --with-openssl=/data/soft/nginx/lnmp1.4-full/src/openssl-1.0.2l
```
{{% /notice %}}

## 5. 参考地址
https://www.cnblogs.com/trustnature/articles/15250071.html

