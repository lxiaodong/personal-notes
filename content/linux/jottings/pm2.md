---
title: "pm2 切换env配置启动不同服务"
tags: 
- pm2
- 随笔
weight: 10
description: "pm2 切换env配置启动不同服务"
---

{{% notice tip %}}
pm2 具体命令参考[官方网址](https://pm2.keymetrics.io/docs/usage/quick-start/)
{{% /notice %}}

## PHP项目根据.env配置启动服务, 启动多服务的情况下，PM2如何处理？
```js
let env = "";
let name = "MessageCenter";
for(let arg of process.argv) {
    if(arg == "--env=aliyun") {
        env = ".env.aliyun";
        name += ":9223";
    }
    if(arg == "--env=ijiwei") {
        env = ".env.ijiwei";
        name += "9224";
    }
}

var exec = require('child_process').exec;
exec('cp ' + env + ' .env', function(error, stdout, stderr) {
    if (error) {
        console.error('error: ' + error);
        return;
    }
})

let config = {
    apps : [
        {
            name        : name,
            script      : "./bin/start.php",
            cwd         : "/home/work/MessageCenter",
            instances   : 1,
            watch       : false,
            ignore_watch: ["runtime", "tmp"],
            max_memory_restart: "150M",
            env_aliyun: {},
            env_ijiwei: {}
        }
    ]
}

module.exports = config;
```

```bash
# 通过如下方式使用
pm2 start messageCenter.config.js --env=ijiwei
[PM2][WARN] Applications MessageCenter9224 not running, starting...
[PM2] App [MessageCenter9224] launched (1 instances)
```