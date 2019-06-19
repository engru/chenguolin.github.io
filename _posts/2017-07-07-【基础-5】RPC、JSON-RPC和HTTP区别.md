---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 基础
---

# 一. RPC
1. RPC是什么
    * RPC(Remote Procedure Call)指的是远程过程调用，简单的说，RPC就是从一台机器上通过参数传递的方式调用另一台机器上的一个函数或方法并得到响应结果。
    * RPC会隐藏底层的通讯细节。
    * RPC是一个请求响应模型，客户端发起请求，服务器返回响应。
    * RPC在使用形式上像调用本地函数一样去调用远程的函数。

2. 常见的RPC框架
    * dubbo: 阿里巴巴公司开源的一个Java高性能优秀的服务框架，使得应用可通过高性能的RPC实现服务的输出和输入功能，可以和Spring框架无缝集成。
    * motan: 新浪微博开源的一个Java 框架。它诞生的比较晚，起于2013年，2016年5月开源。Motan 在微博平台中已经广泛应用，每天为数百个服务完成近千亿次的调用。
    * rpcx: Go语言生态圈的Dubbo，比Dubbo更轻量，实现了Dubbo的许多特性，借助于Go语言优秀的并发特性和简洁语法，可以使用较少的代码实现分布式的RPC服务。
    * gRPC: Google开发的高性能、通用的开源RPC框架，主要面向移动应用开发并基于HTTP2协议标准而设计，基于ProtoBuf(Protocol Buffers)序列化协议开发，且支持众多开发语言。
    * thrift: Apache的一个跨语言的高性能的服务框架
    * JSON-RPC: JSON-RPC是一个无状态且轻量级的远程过程调用(RPC)协议。
    
3. 实现RPC框架  
   由于RPC使用形式上调用同一个进程内存空间的函数或方法一样，因此需要解决以下3个问题
    * `寻址`: 客户端调用的时候怎么告诉服务端调用的是哪个函数或方法，在RPC框架中每个函数都有自己的CallID，所以客户端在调用的时候需要带上这个CallID表示调用哪个函数。CallID可以使用函数字符串名称，也可以使用整数(需要有一个映射表来关联)。
    * `序列化反序列化`: 由于客户端和服务端不是同一个进程不能通过内存来传递参数，因此需要客户端先把参数序列化成字节流传给服务端，服务端收到字节流后反序列为自己能读取的格式，序列化反序列可以使用Protobuf、JSON等。
    * `网络传输`: 客户端和服务端需要通过网络连接来传输数据，因此需要有一个网络的传输层。网络传输可以使用Socket、TCP、UDP、HTTP、HTTP2等。

# 二. HTTP
1. HTTP请求本身也可以看做是RPC的一种具体形式。HTTP请求也一样是可以从客户端发一个信号到服务端，服务端上执行某个函数，然后返回一些信息给客户端。HTTP请求非常常见，如果我们自己想开放自己机器的部分功能给任意的人用，那么使用HTTP API的形式是非常合适的，因为HTTP请求是大家都经常使用的方式。而很多时候，对于公司内部的两台机器之间，大家会按照实际需要去自定制一套RPC，这样做的好处是灵活高效，但是坏处就是没有通用性。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/RPC-HTTP.jpeg?raw=true)

2. HTTP和RPC异同
    * HTTP请求和RPC的相同点是同样都具有请求和响应，二者的基本过程是一样的，首先从客户端发出请求，服务端收到之后执行某段代码，然后把运算结果或者是报错信息作为响应，返回给客户端。
    * HTTP请求往往围绕资源，而RPC的请求往往围绕一个动作。比如一个常见的HTTP请求有`GET index`，`POST posts`分别表示请求首页，或者发布一篇文章。而用RPC执行相同的任务，就是指定一个函数`get_index`或者`create_post` 。
    * HTTP请求的服务器上，一般需要安装Nginx或者Apache这样的HTTP服务器软件，而提供RPC的服务器不一定需要安装这些软件。

# 三. JSON-RPC
1. `JSON-RPC`是一个无状态且轻量级的RPC协议，其传输内容以`JSON`方式，相对于一般的HTTP请求通过URI调用远程服务器，JSON-RPC直接在内容中定义了要调用的函数名称（如 {"method": "getUser"}），对于开发者来说非常的方便。`Bitcoin`和`Ethereum`都支持JSON-RPC通过客户端直接调用节点上的函数或方法。
   `注意: 以rpc开头的方法名预留作为系统扩展，且必须不能用于其他地方。`

2. JSON-RPC请求  
   JSON-RPC 2.0和1.0之间一些差异，我们这里介绍2.0的使用，一个JSON-RPC的请求必须包含以下4个字段。
    * `jsonrpc`: 指定JSON-RPC的版本，必须设置为2.0
    * `id`: 调用标识符，用于标示一次远程调用过程，值必须包含一个字符串、数值。
    * `method`: 所要调用方法名称的字符串
    * `params`: 方法传入的参数，若无参数则传入空`[]`

3. JSON-RPC响应  
   当发起一个RPC调用时，除通知之外服务端都必须有响应，响应表示为一个JSON对象包含以下几个字段。
    * `jsonrpc`: 指定JSON-RPC的版本，固定为为2.0
    * `id`: 调用标识符，用于标示一次远程调用过程，值必须包含一个字符串、数值。
    * `result`: 如果调用成功则显示响应结果
    * `error`: 如果调用失败则显示错误的信息，error带有以下几个字段
        * `code`: 错误类型，必须为整数 【必须】
        * `message`: 错误的简单描述字，该描述应尽量简短 【必须】
        * `data`: 包含关于错误附加信息的基本类型或结构化类型 【可选】
    
4. JSON-RPC错误码
    * `-32700`: Parse error语法解析错误 (服务端接收到无效的json。该错误发送于服务器尝试解析json文本)
    * `-32600`: Invalid Request (发送的json不是一个有效的请求对象)
    * `-32601`: Method not found (该方法不存在或无效)
    * `-32602`: Invalid params  (无效的方法参数)
    * `-32603`: Internal error (JSON-RPC内部错误)
    * `-32000 ~ -32099`: Server error (预留用于自定义的服务器错误)
    * `-32768 ~ -32000`: 保留的预定义错误代码, 保留下列以供将来使用

5. JSON-RPC示例

   ```
   1. 带索引数组参数的rpc调用
   --> {"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}
   <-- {"jsonrpc": "2.0", "result": 19, "id": 1}
   
   2. 带关联数组参数的rpc调用
   --> {"jsonrpc": "2.0", "method": "subtract", "params": {"subtrahend": 23, "minuend": 42}, "id": 3}
   <-- {"jsonrpc": "2.0", "result": 19, "id": 3}
   
   3. rpc批量调用
   --> [{"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
        {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
        {"jsonrpc": "2.0", "method": "subtract", "params": [42,23], "id": "2"},
        {"foo": "boo"},
        {"jsonrpc": "2.0", "method": "foo.get", "params": {"name": "myself"}, "id": "5"},
        {"jsonrpc": "2.0", "method": "get_data", "id": "9"}]
   <-- [{"jsonrpc": "2.0", "result": 7, "id": "1"},
        {"jsonrpc": "2.0", "result": 19, "id": "2"},
        {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
        {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "5"},
        {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}]
   ```

   ```
   1. 不包含调用方法的rpc调用
   --> {"jsonrpc": "2.0", "method": "foobar", "id": "1"}
   <-- {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "1"}
   
   2. 包含无效json的rpc调用
   --> {"jsonrpc": "2.0", "method": "foobar, "params": "bar", "baz]
   <-- {"jsonrpc": "2.0", "error": {"code": -32700, "message": "Parse error"}, "id": null}
   
   3. 无效请求对象的rpc调用
   --> {"jsonrpc": "2.0", "method": 1, "params": "bar"}
   <-- {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
   ```

