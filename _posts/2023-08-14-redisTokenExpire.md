---
layout: post
title: ruoyi框架里面设置登录超时时间不能生效
categories: redis
description: redis过期策略
keywords:  redis
---

## 问题

项目使用到了ruoyi框架，给登录超时限制了12小时，但是每次几个小时后就提示过期，要求再登录一次。

首先怀疑是不是redis内存淘汰策略出现了问题。

先使用一下的命令

```
systemctl status redis
```

可以查询到redis loaded所在地方

![image-20230814093251481](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230814093251481.png)

根据上述redis loaded输入

```
cat /lib/systemd/system/redis-server.service
```

![image-20230814094020903](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230814094020903.png)

可以看到ExecStart，根据这个查到conf的所在位置，vim conf文件 修改maxmemory-policy为allkeys-lru

保存后用

```
redis-server /etc/redis/redis.conf
```

输入

```
redis-cli
```

再输入

```
CONFIG GET maxmemory-policy
```

可以看到当前的policy已经发生了改变

![image-20230814094201525](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230814094201525.png)

