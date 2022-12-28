---
title: "Nginx lua模块引用cjson"
tags: 
- nginx
- lua
- cjson
weight: 2
---

## 1. 问题
lua脚本需要解析一个请求过来的json格式的数据，原有的luajit并没有带cjson库，需要自己手动安装一下。

## 2. 安装

#### 2.1下载安装LuaJIT-2.0.4.tar.gz，需配置环境变量
{{% notice tip %}}
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

#### 2.2 下载安装cjson
```bash
    cd /data/soft/nginx

    wget https ://github.com/Kong/kong-cjson/archive/2.1.0.tar.gz

    tar zxvf 2.1.0.tar.gz

    cd kong-cjson-2.1.0/

    make && make install
```

## 3. 可能遇到的问题
上述安装过程是已经解决了所有问题的方法，以下是可能遇到的问题？

{{% notice info %}}
#### 3.1 No such file or directory
报错信息如下：
```text
    lua_cjson.43:17:  lua. No such file or directory
    lua_cjson.44:21:  lauxlib. No such file or directory
    lua_cjson.192:  expected ‘)’ before ‘*’ token
    lua_cjson.206:  expected ‘)’ before ‘*’ token
    lua_cjson.218:  expected ‘)’ before ‘*’ token
    lua_cjson.237:  expected ‘)’ before ‘*’ token
    lua_cjson.266:  expected ‘)’ before ‘*’ token
    lua_cjson.279:  expected ‘)’ before ‘*’ token
    lua_cjson.288:  expected ‘)’ before ‘*’ token
    lua_cjson.296:  expected ‘)’ before ‘*’ token
    lua_cjson.304:  expected ‘)’ before ‘*’ token
    lua_cjson.336:  expected ‘)’ before ‘*’ token
```
解决方法：修改Makefile中LUA_INCLUDE_DIR，这个引用地址会找安装的luajit安装目录
```text
    cd /data/soft/nginx/kong-cjson-2.1.0

    vim Makefile
    ##### Build defaults #####
    LUA_VERSION =       5.1
    TARGET =            cjson.so
    PREFIX =            /usr/local
    #CFLAGS =            -g -Wall -pedantic -fno-inline
    CFLAGS =            -O3 -Wall -pedantic -DNDEBUG
    CJSON_CFLAGS =      -fpic
    CJSON_LDFLAGS =     -shared 
    LUA_INCLUDE_DIR =   $(PREFIX)/luajit/include/luajit-2.0/
    LUA_CMODULE_DIR =   $(PREFIX)/lib/lua/$(LUA_VERSION)
    LUA_MODULE_DIR =    $(PREFIX)/share/lua/$(LUA_VERSION)
    LUA_BIN_DIR =       $(PREFIX)/bin

    make && make install #就会看到成功了
```
{{% /notice %}}