---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

# 一. 介绍
`HTTP/HTTPS API`接口是互联网各系统之间对接的重要方式之一，使用HTTP/HTTPS API接口开发和调用都很方便，也是被大量采用的方式，它可以让不同系统之间实现数据的交换和共享，最常用的HTTP操作类型主要是 `GET` 和 `POST`。

常用于以下3个场景

1. `客户端->服务端`: 客户端(Android/iOS/Web)通过公网使用`HTTPS`请求服务API接口做数据交互
2. `服务A->服务B (公网)`: 服务通过公网使用`HTTPS`请求另外一个服务API接口做数据交互
3. `服务A->服务B (内网)`:服务通过内网使用`HTTP`请求另外一个服务API接口做数据交互

针对以上3个使用场景，结合公有云的一些使用姿势，总结出这3种场景下 HTTP/HTTPS API接口设计指南。

# 二. 设计指南
## ① 接口风格
API接口应该使用 [RESTful API](https://en.wikipedia.org/wiki/Representational_state_transfer) 风格，并遵循以下几个设计原则

API的URI scheme 由以下几个部分组成: {URI Scheme} :// {Endpoint} / {Version} /  {Resource Path} ? {Query string}

1. `URI Scheme`: 传输请求的协议，走公网所有API应该使用`HTTPS`协议，走内网所有API可以使用`HTTP`协议  【必须】
2. `Endpoint`: 承载REST服务端点的服务器域名或IP，例如`www.xxx.com`  【必须】
3. `Version`: API版本，例如 `v1`   【必须】
4. `Resource Path`: API访问路径，例如 `/auth/tokens` 表示获取用户Token，Path一般使用复数名词表示  【必须】
5. `Query String`: 查询参数，参数格式为 `参数名=参数值`多个参数使用`&`拼接【可选】

举例: `https://www.huaweicloud.com/v1/auth/tokens` 或 `https://www.huaweicloud.com/v1/kafka/topics?instance_id=xxxx`

## ② 请求消息头
请求消息Headers用来设置附加字段，例如指定的URI和HTTP方法所要求的字段。`Content-Type` 字段是必须的，用来定义消息体类型

请求方发起请求的时候必须设置 `Content-Type` 值为 `application/x-www-form-urlencoded; charset=utf-8`

## ③ 请求参数
我们把请求参数分成2类，一类是`公共参数`，另外一类是`业务参数`，参数传递应该遵循以下2个原则

1. 无论是`GET`还是`POST`请求，参数应该使用 `key=value`并使用`&`连接这种方式
2. 对于`GET`请求，所有的参数应该使用 `key=value`并使用`&`连接，做为URI的Query string
3. 对于`POST`请求，公共参数应该使用 `key=value`并使用`&`连接，做为URI的Query string。业务参数使用`key=value`并使用`&`连接，做为data数据

正常情况下，请求参数还需要进行特殊字符转义，可以参考 [url Encode/Decode](https://github.com/chenguolin/go-common/blob/master/url/url.go)

常见的`公共参数`如下，不同场景需要的参数不同

| 名称 | 类型 | 使用场景 | 描述 |
|:--|:--|:--|:--
| os_type  | string | 客户端->服务端 | 客户端操作系统类型，Android、iOS |
| os_version  | string | 客户端->服务端 | 客户端操作系统版本 |
| app_version  | string | 客户端->服务端 | 客户端版本 |
| network  | string | 客户端->服务端 | 客户端网络类型，例如2G、3G、4G、5G、wifi |
| mac_addr  | string | 客户端->服务端 | 客户端Mac地址 |
| language  | string | 客户端->服务端 | 客户端语言类型，常用于多语言场景 <br> `EN`标识英文，`ZhHans`标识中文 |
| signature  | string | 通用 | 签名结果 |
| signature_version  | string | 通用 | 签名版本，例如`1.0` |
| signature_nonce  | string | 通用 | 签名唯一随机数 <br> 用于防止网络重放攻击，建议您每一次请求都使用不同的随机数 |
| sig_timestamp  | string | 通用 | 签名时间戳，服务端会校验是否过期 |

## ④ 响应结果
API接口响应结果应该按照以下格式

```
{
  "meta": {
      "code": 0,
      "message": "",
      "error": ""
  },
  "response": {
      ......
  }
}
```

1. meta: 包含错误码，错误信息
   + code: 错误码，当值为0时表示成功，其他值表示不同类型的错误；
   + message: 用户友好提示错误信息
   + error: 程序内部错误信息，用于给开发排错用
2. response: 包含本次请求获取到的相关数据，有些情况可能为空

# 三. 接口安全
由于API接口开放在互联网上，所以我们就需要有足够的安全措施来保证接口安全，之前写过2篇文章 [简单的http签名算法](https://chenguolin.github.io/2017/07/25/HTTP-API-1-%E7%AE%80%E5%8D%95%E7%9A%84http%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95/) 和 [http api接口安全性设计](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/)，可以先简单了解一下。

上述提到的3个场景，接口安全性要求是不同的，针对这3个场景，我总结了接口安全设计方法

## ① 客户端->服务端
用户客户端调用服务端是最常见的场景，我们可以把API接口分成以下几类

1. 通用接口: 和用户无关接口，用户无须登录即可访问，这类接口对接口安全性要求弱
2. 用户相关接口: 和用户强相关，用户需要登录才可以访问，这类接口安全性要求高，必须要保证用户请求数据不被篡改

针对这个场景，需要做以下几点来保证接口安全性

1. 使用`HTTPS`代替`HTTP`，使用HTTPS加密传输，避免明文传输导致数据泄露或被劫持篡改
2. 使用`私钥签名公钥验签`方式来保证请求参数不会被窜改，具体可以参考 [HTTP API接口安全性设计 - 私钥签名公钥验签](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/#%E4%B8%89-%E7%A7%81%E9%92%A5%E7%AD%BE%E5%90%8D%E5%85%AC%E9%92%A5%E9%AA%8C%E7%AD%BE)

## ② 服务A->服务B (公网)

## ③ 服务A->服务B (内网)

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

需要注意的是，开发者绝不能将AccessKeySecret密钥包含在分发给最终用户的程序中，无论是包含在配置文件中还是二进制文件中都会带来非常大的密钥泄漏风险。

消息响应的标准格式？
{
    "meta":{...},
    "response": {...}
}

安全建议
1. 不要将AccessKey嵌入代码，应该使用数据库或者写文件方式，避免泄露
2. 定期更新AccessKey，保证一些旧的代码泄漏后不会影响现有的业务



