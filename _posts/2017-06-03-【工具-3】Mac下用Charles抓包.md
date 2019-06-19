---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 工具
---

Mac os APP Charles抓包使用指南

# 一. 下载Charles v4.2 并安装
1. 云盘下载：https://pan.baidu.com/s/1kUSBpCb 
2. 安装后先打开Charles一次
3. 下载破解文件 charles.jar  
   下载地址：http://charlesproxy.iiilab.com/4_2/charles.jar
4. 替换掉原文件夹里的charles.jar  
   路径地址: /Applications/Charles.app/Contents/Java/charles.jar
5. 完成

# 二. HTTP抓包
1. 在Mac中打开Charles应用
2. 设置手机HTTP代理  
   手机与Mac笔记本在同一局域网内 手机添加代理ip地址（Mac内网地址）和端口号（8888）
3. 在手机上访问App，在Mac Charles应用弹出的确认窗中选择Allow，允许即可

# 三. HTTPS抓包
1. Mac安装Charles证书  
   Charles -> Help ->  SSL Proxying -> Install Charles Root Certificate  
   弹出窗口选择`添加`
2. Mac Charles证书信任
   打卡`钥匙串`应用，搜索`Charles`双击修改`信任`，改为`始终信任`
3. 手机安装Charles证书  
   Charles -> Help ->  SSL Proxying -> Install Charles Root Certificate On a Mobile Device Or ...  
   弹窗提示用户使用手机访问`https://chls.pro/ssl`，之后便会自动下载安装证书到手机
4. 手机设置信任证书  
   设置 -> 通用 -> 关于本机 -> 证书信任设置 -> 选择信任
5. charles设置SSL proxy setting  
   Charles -> Proxy -> SSL Proxying Settings
   1). Enable SSL Proxying
   2). Add: host: *; port: 443

# 四. 常见问题
1. 如果报`you may need to configure your browser or application to trust the charles root certificate. see ssl proxying in the help menu.`，是因为Mac或者手机证书过期或者未信任，可以按照以下方法来排查  
   1). Mac访问`https://baidu.com`，如果不能访问说明可能是证书过期，如果能访问则说明证书有效  
   2). 手机访问`https://baidu.com`，如果不能访问说明可能是证书过期，如果能访问则说明证书有效


