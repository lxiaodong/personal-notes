---
title: "substr"
tags: 
- php function
- 随笔
weight: 10
description: "substr function 使用细节"
---

## 1. substr 细节
```bash
$ php -r '$a = 123456; echo substr($a, 0, 3)."\n";'
123
```

```bash
$ php -r '$a = 123456; echo substr($a, 3, -1)."\n";'
45

```

{{% notice tip %}}
查看[PHP](https://www.php.net/manual/zh/function.substr)官方文档，可以知道如果获取完整的后续字符串, `length`可以传null, 如下。
{{% /notice %}}

```bash
$ php -r '$a = 123456; echo substr($a, 3)."\n";'
456

```