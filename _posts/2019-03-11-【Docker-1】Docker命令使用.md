---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

https://mp.weixin.qq.com/s/jv480Q0g6KGKMJhqe0tbjQ

1. 监听docker daemon进程事件流(Docker daemon会监听Unix域套接字：/var/run/docker.sock)  
   curl --unix-socket /var/run/docker.sock http://localhost/events
