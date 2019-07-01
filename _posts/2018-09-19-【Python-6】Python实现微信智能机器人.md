---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

# 一. 介绍
[py_weixin_robot](https://github.com/chenguolin/script/tree/master/py_weixin_robot) 是用Python实现的一个智能微信机器人，后端采用图灵机器人接口来实现智能问答。   
   
要求和限制
1. Python 2.7 版本及以上
2. 需要申请[图灵机器人](http://www.turingapi.com/) 获取ApiKey
3. 目前只支持文本消息回复

消息发送机制
1. 机器人收到消息后会先判读是否发送和接收都是当前用户，如果是则智能回答。否则判读用户是否在2分钟内操作过手机，如果有则跳过，否则进行回答判断
2. 如果消息来自群聊，先判断当前消息是否 @当前用户，如果没有则随机选择是否发送消息
3. 如果消息来自一般用户，消息类型是图片则随机回复，消息类型是文本则调用图灵机器人接口获取智能回答然后发送消息
4. 如果消息来自公众号，直接过滤

# 二. 使用说明
[py_weixin_robot](https://github.com/chenguolin/script/tree/master/py_weixin_robot) 项目源码已经开源，欢迎交流学习

1. python weixin_rebot.py
2. 扫描二维码登录即可  (目前微信限制web版和PC客户端没有办法同时登录)

安装依赖
```
1. linux: 
   1). ubuntu 12.04
   2). sudo apt-get install imagemagick
       sudo apt-get install python-pip
2 python:
   1). version > 2.6
   2). sudo pip install requests
   3). sudo pip install pypng
   4). sudo pip install Pillow
```

# 三. web微信登录过程
## ① 获取uuid  
1. 说明: 微信Web版本不使用用户名和密码登录，而是采用二维码登录，所以服务器需要首先分配一个唯一的会话ID，用来标识当前的一次登录
2. api: https://login.wx.qq.com/jslogin?appid=wx782c26e4c19acffb&redirect_uri=https%3A%2F%2Fwx.qq.com%2Fcgi-bin%2Fmmwebwx-bin%2Fwebwxnewloginpage&fun=new&lang=zh_CN&_=%s
3. `GET`请求
4. 参数说明
    * `appid`: 固定值`wx782c26e4c19acffb`表示微信网页版  
    * `redirect_uri`: 固定值`https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage`
    * `fun`: 固定值`new`
    * `lang`: 固定值`zh_CN`表示中文  
    * `_`: 表示13位时间戳  

## ② 获取登录二维码
1. 说明: get请求拿到数据，再保存为图片并展示
2. api: https://login.weixin.qq.com/qrcode/%s
3. `GET`请求
4. 参数说明
    * uuid: 当前用户uuid

## ③ 扫描二维码等待用户确认
1. 说明: 当用户拿到二维码数据之后，用户需要扫描二维码并点击确认登录
2. api: https://login.weixin.qq.com/cgi-bin/mmwebwx-bin/login?loginicon=true&uuid=%s&tip=0&r=-979422099&_=%s
3. `GET`请求
4. 参数说明  
    * `uuid`: 表示当前uuid  
    * `tip`: 0->表示等待用户确认登录。返回window.code=200;window.redirect_uri="https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage?ticket=AWskQEu2O8VUs7Wf2TVOH8UW@qrticket_0&uuid=IZu2xb_hIQ==&lang=zh_CN&scan=1467393709"; 表示登录成功  
    * `r`: -979422099 随机9位字符串即可  
    * `_`: 表示当前13位时间戳  
5. 点击登录成功之后，返回redirect_uri表示用户已经在手机端完成了授权过程，需要继续访问当前链接获取wxuin和wxsid  

## ④ 访问登录地址，获得uin和sid
1. 说明: wxuin和wxsid在后续的通信中都需要用到
2. api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage?ticket=AWskQEu2O8VUs7Wf2TVOH8UW@qrticket_0&uuid=%s==&lang=zh_CN&scan=%s
3. `GET`请求
4. 参数说明
    * `uuid`: 表示当前uuid  
    * `scan`: 表示当前的扫描时间戳 10位  
5. 返回结果  
   ```
   <error><ret>0</ret><message>OK</message><skey>@crypt_b13bcf4_b49e9a98c34a5e263381f032e6ae18cb</skey><wxsid>gJmKzszzW2KnFIXf</wxsid><wxuin>828185640</wxuin><pass_ticket>vBTQ11xy5sLBraR8BT1K7xGQldAmLRSZBzylJ%2FiEVbV77SgMcNyOBj0seNho8kUM</pass_ticket><isgrayscale>1</isgrayscale></error>
   ```
   1). skey  
   2). wxsid  
   3). wxuin --> 用户唯一的id  
   4). pass_ticket  

## ⑤. 初始化登录页信息
1. 说明: 初始化登录后页面的信息，获取常用联系人以及微信公众号
2. api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxinit?r=-%s&lang=zh_CN&pass_ticket=%s
3. `POST`请求
4. Post请求data格式
   ```
   {  
      'BaseResponse': {  
          'Uin': wxuin,  
          'Sid': wxsid,  
          'Skey': skey,  
          'DeviceID': //随机串 'e'+str(random.random())[2:17]  
      }  
   }
   ```
5. 返回json串包含当前user的相关信息，BaseResponse内Ret位0表示成功
   
## ⑥. 开启微信状态通知
1. 说明: 有新消息通知的时候会显示
2. api:  https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxstatusnotify?lang=zh_CN&pass_ticket=%s
3. `POST`请求
4. Post请求data格式
   ```
   {  
      'BaseResponse': {  
          'Uin': wxuin,  
          'Sid': wxsid,  
          'Skey': skey,  
          'DeviceID': //随机串 'e'+str(random.random())[2:17]  
      }  
      'ClientMsgId': 13位时间戳,  
      'Code': 3  //固定值  
      'FromUserName':  userNmae, //从初始化登录信息那边取到  
      'ToUserName': userNmae, //从初始化登录信息那边取到  
   } 
   ```
5. 返回json串，BaseResponse内Ret位0表示成功  
   
## ⑦. 同步刷新，检查服务端消息  
1. 说明: 为了实时取到消息，需要客户端不断发送请求给服务端查询  
2. api: https://webpush.wx.qq.com/cgi-bin/mmwebwx-bin/synccheck?r=%s&skey=%s&sid=%s&uin=%s&deviceid=%s&synckey=%s&_=%s  
3. `GET` 请求  
4. 参数说明  
    * `r`: 13位时间戳  
    * `b`: skey 同上,需要url quote  
    * `c`: sid 同上  
    * `d`: devicedid 同上  
    * `e`: synckey由 初始化登录页面信息返回串中Sync的list列表组成, 需要url quote  
    * `f`: _ 13位时间戳  
 5. 返回值 `window.synccheck={retcode:"xxx",selector:"xxx"}`
    ```
    retcode:  
    1). 0 正常  
    2). 1100 失败/登出微信  
    3). 1101 从其它设备登录微信网页版  
       
    selector:  
    1). 0 正常  
    2). 2 新的消息  
    3). 7 手机操作了微信
    ```
       
## ⑧. 获取消息
1. 说明: 根据同步刷新的接口拿到当前的状态后，如果是新的消息，需要获取新的消息
2. api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxsync?sid=%s&skey=%s&pass_ticket=%s
3. `GET`请求
4. 返回json串
    * BaseResponse，Ret位0表示返回成功  
    * AddMsgCount 表示新消息个数  
    * AddMsgList 表示新消息列表，MsgType说明  
        * 1: 文本消息  
        * 3: 图片消息  
        * 34: 语音消息  
        * 37: VERIFYMSG  
        * 40: POSSIBLEFRIEND_MSG  
        * 42: 共享名片  
        * 43: 视频通话消息  
        * 47: 动画表情  
        * 48: 位置消息  
        * 49: 分享链接  
        * 50: VOIPMSG  
        * 51: 微信初始化消息  
        * 52: VOIPNOTIFY  
        * 53: VOIPINVITE  
        * 62: 小视频  
        * 9999: SYSNOTICE  
        * 10000: 系统消息  
        * 10002: 撤回消息  

## ⑨ 发送消息
1. 说明: 发送数据给指定用户，包括群聊等
2. api: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxsendmsg
3. `POST` 请求
4. Post请求data格式
   ```
   {  
       'BaseResponse': {  
           'Uin': wxuin,  
           'Sid': wxsid,  
           'Skey': skey,  
           'DeviceID': //随机串 'e'+str(random.random())[2:17]  
       }  
       'Type': //消息类型，同上  
       'Content': //消息内容  
       'FromUserName':  //发送用户  
       'ToUserName': //接受用户  
       'LocalID':  //13位时间戳+4位随机数  
       'ClientMsgId': //同LocalId  
   }
   ```
5. 返回json串，BaseResponse内Ret位0表示成功  

