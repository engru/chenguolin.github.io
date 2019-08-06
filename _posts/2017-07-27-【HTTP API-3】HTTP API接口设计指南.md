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

1. 必须设置 `Content-Type: application/x-www-form-urlencoded; charset=utf-8`
2. 对于需要用户认证信息的接口，需要设置 `Access-Token: {token}` 用来标识某个用户

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
用户客户端通过`公网`调用服务端是最常见的场景，目前我们能看到的任何一个APP都会涉及到API接口调用，我们可以把API接口分成以下2类

1. 通用接口: 和用户无关接口，用户无须登录即可访问，这类接口对接口安全性要求弱
2. 用户相关接口: 和用户强相关，用户需要登录才可以访问，这类接口安全性要求高，必须要保证用户请求数据不被篡改

针对这个场景，需要做以下几点来保证接口安全性

1. 使用`HTTPS`代替`HTTP`，使用HTTPS加密传输，避免明文传输导致数据泄露或被劫持篡改
2. 客户端使用`私钥签名公钥验签`方式来保证请求参数不会被窜改，具体可以参考 [HTTP API接口安全性设计 - 私钥签名公钥验签](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/#%E4%B8%89-%E7%A7%81%E9%92%A5%E7%AD%BE%E5%90%8D%E5%85%AC%E9%92%A5%E9%AA%8C%E7%AD%BE)
3. 用户相关的接口，客户端需要通过`token认证`来标识某个用户  (通常在请求消息头添加`Access-Token: {token}`)
4. 服务端使用公钥验证签名，需要保证sig_time在有效期内同时signature_nonce没有被使用过

`客户端会持有私钥，因此需要保证私钥不能够泄露，如果泄露需要及时替换新的公私钥对`

## ② 服务A->服务B (公网)
服务A通过`公网`调用服务B，在公有云的场景下比较场景，例如使用阿里云上的Kafka服务，我们在自己的机房通过公网访问阿里云Kafka服务创建Topic、查看Topic列表。

针对这个场景，公有云厂商一般是使用以下方式来保证接口安全性

1. 使用`HTTPS`代替`HTTP`，使用HTTPS加密传输，避免明文传输导致数据泄露或被劫持篡改
2. 服务A 使用`AK/SK`签名认证 (常使用HMAC-SHA签名)
   AK/SK认证就是使用AK/SK对请求进行签名，在请求时将签名信息添加到Query string，从而通过身份认证。  
   AK(Access Key ID): 访问密钥ID，与私有访问密钥一起使用，对请求进行加密签名。  
   SK(Secret Access Key): 私有访问密钥，与访问密钥ID结合使用对请求进行加密签名，可标识发送方，并防止请求被修改。  
3. 服务B 使用相同的算法计算一次签名，如果签名一致说明请求没有被篡改，同时需要保证sig_time在有效期内同时signature_nonce没有被使用过
   
安全建议

1. 不要将`AK/SK`嵌入代码，应该使用数据库或者写文件方式，避免泄露
2. 定期更新`AK/SK`，保证一些旧的代码泄漏后不会影响现有的业务

阿里云AK/SK签名算法可以参考 [calculate signature](https://github.com/chenguolin/go-aliyun/blob/master/kafka/client.go#L289)

## ③ 服务A->服务B (内网)
服务A通过`内网`调用服务B，常用于服务间依赖调用，这在我们自己的机房非常常见。

由于是内网调用，安全性会比在公网上调用高很多，因此为了接口安全性我们一般只需要使用简单的签名就可以，具体可以参考 [HTTP API接口安全性设计 - 简单参数签名](https://chenguolin.github.io/2017/07/26/HTTP-API-2-HTTP-API%E6%8E%A5%E5%8F%A3%E5%AE%89%E5%85%A8%E6%80%A7%E8%AE%BE%E8%AE%A1/#%E4%BA%8C-%E5%8F%82%E6%95%B0%E7%AD%BE%E5%90%8D)

