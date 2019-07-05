---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

设计点
1. REST风格？

安全性
1. 启用HTTPS
2. 启用签名鉴权
    * 简单sig签名
    * 公私钥签名
3. 服务端检查参数合法性
4. 公有云鉴权方式参考
   1). 阿里云: https://help.aliyun.com/document_detail/122301.html?spm=a2c4g.11186623.6.602.8c2951aeFN3veF  
   2). 华为云: https://support.huaweicloud.com/api-dms/zh-cn_topic_0086485751.html （鉴权方面参考）  
5. token认证？和 AK/SK认证区别？
