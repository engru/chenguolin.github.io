---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

# 一. 

# 二. 
设计点
1. REST风格？REST（Representational State Transfer）

安全性
1. 启用HTTPS
2. 启用签名鉴权
    * 简单sig签名
    * 公私钥签名
3. 服务端检查参数合法性
4. 公有云鉴权方式参考  
   1). 阿里云: https://help.aliyun.com/document_detail/122301.html?spm=a2c4g.11186623.6.602.8c2951aeFN3veF  
   2). 华为云: https://support.huaweicloud.com/api-dms/zh-cn_topic_0086485751.html （鉴权方面参考）  
   3). API签名: https://support.huaweicloud.com/devg-apisign/api-sign-provide.html
5. AK/SK认证？ 推荐使用  （还有token认证方式）


内网调用接口设计？  
用户客户端走公网接口设计？

接口类型
1. 通用请求接口，类似Get xxx列表，不需要传参数，安全性会稍微弱一点
2. 涉及用户信息接口，需要上传相关的凭证
3. 涉及用户上报参数接口，安全性很高


token认证：
Token在计算机系统中代表令牌（临时）的意思，拥有Token就代表拥有某种权限。Token认证就是在调用API的时候将Token加到请求消息头，从而通过身份认证，获得操作API的权限。
在构造请求中以调用获取用户Token接口为例说明了如何调用API。获取Token后，再调用其他接口时，您需要在请求消息头中添加“X-Auth-Token”，其值即为Token。例如Token值为“ABCDEFJ....”，则调用接口时将“X-Auth-Token: ABCDEFJ....”加到请求消息头即可，如下所示。

AK/SK认证
AK/SK认证就是使用AK/SK对请求进行签名，在请求时将签名信息添加到消息头，从而通过身份认证。
AK(Access Key ID)：访问密钥ID。与私有访问密钥关联的唯一标识符；访问密钥ID和私有访问密钥一起使用，对请求进行加密签名。
SK(Secret Access Key)：与访问密钥ID结合使用的私有访问密钥，对请求进行加密签名，可标识发送方，并防止请求被修改。

是用户访问内部资源最重要的身份凭证。用户调用API时的通信加密和身份认证会使用API凭证（即基于非对称密钥算法的鉴权密钥对）。API凭证是云上用户调用云服务API、访问云上资源的唯一身份凭证。
在阿里云，用户可以使用AccessKey（简称AK）构造一个API请求（或者使用云服务SDK）来操作资源。AccessKey包括AccessKeyId和AccessKeySecret。其中AccessKeyId用于标识用户，AccessKeySecret是用来验证用户身份合法性的密钥。AccessKeySecret必须保密。

消息响应的标准格式？
{
    "meta":{...},
    "response": {...}
}

安全建议
1. 不要将AccessKey嵌入代码，应该使用数据库或者写文件方式，避免泄露
2. 定期更新AccessKey，保证

