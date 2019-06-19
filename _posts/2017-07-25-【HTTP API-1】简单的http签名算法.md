---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

常见的CS架构中，客户端经常需要请求服务端提供的公网接口。由于接口暴露在公网，存在以下几个问题

1. 接口就很容易被其他人利用甚至有可能导致被刷接口影响服务可用性。
2. 请求被篡改，影响实际请求结果
3. 其它问题

为了解决这些问题，最简单的方法是客户端请求服务端的时候加个sig签名，服务端对sig签名做个判断。  
通过简单的sig签名能够防住一些被刷接口的可能，同时也能够做下请求校验，可以达到初步的防御效果。

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import sys
import time
import md5

# 常量定义
SIG = "sig"
SIG_TIME = "sigTime"
SIG_VERSION = "sigVersion"
SIG_FINAL_STRING = "sigFinalString"
SIG_SDK_APP_SECRET = "xxx"              # 密钥 客户端服务端提前约定好即可
SIG_SDK_ADD_KEY = "yyy"                 # 盐   客户端服务端提前约定好即可

def get_signature(path, params, sig_time):
    new_path = path.lstrip("/")
    new_params = sort_params(params)
    return generate_signature(new_path, new_params, sig_time)

def sort_params(params):
    values = list()
    for k, v in params.items():
        if k == SIG or k == SIG_TIME or k == SIG_VERSION or k == SIG_FINAL_STRING:
            continue
        values.append(str(v))
    return sorted(values)

def generate_signature(path, params, sig_time):
    content = path + "".join(params) + SIG_SDK_APP_SECRET + str(sig_time) + SIG_SDK_ADD_KEY
    sig = md5.new()
    sig.update(content)

    # md5 打散
    hexSig = sig.hexdigest()
    newHexSig = ""
    for i in range(16):
        i = i * 2
        newHexSig += hexSig[i+1]
        newHexSig += hexSig[i]
    return newHexSig

if __name__ == "__main__":
    # TODO 写死参数
    path = "/get/username.json"
    params = {
            "id": 1234567890,
            "status": 1,
            "value": "100000000000000000",
            "name": "chenguolin"
    }
    sig_time = 1539847107000
    # call get_signature
    print get_signature(path, params, sig_time)
```

1. 去掉url path中左边的/
2. 参数排序
3. 所有的参数和密钥、盐拼成一个字符串
4. 字符串计算一个md5
5. md5字节打散


