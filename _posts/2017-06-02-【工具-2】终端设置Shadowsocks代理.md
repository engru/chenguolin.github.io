---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 工具
---

# 一. Shadowsocks原理
Shadowsocks(简称SS)是目前最好的科学上网方式，它的流量经过加密，所以没有流量特征不会被GFW防火墙拦截。  
下面简单介绍一下SS的原理

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/what-is-shadowsocks.png?raw=true)

SS要求本机运行SS Local服务，海外服务器运行SS Server服务。SS Local服务默认监听localhost的1080端口，该端口代理浏览器的请求。

1. 本地浏览器发出基于Socks5协议请求，请求先到SS Local服务
2. SS Local服务对流量加密转成普通TCP包，成功穿过GFW防火墙，然后把普通的TCP流量发往海外SS Server服务
3. SS Server将收到的加密数据进行解密，还原初始请求，再发送到用户需要访问的服务网站获取响应结果
4. SS Server对响应结果进行了加密转成普通TCP包，成功穿过GFW防火墙
5. SS Local收到加密结果后进行解密返回给浏览器

`SS Local和SS Server使用用户设置的密码和加密算法进行加解密`

# 二. 终端SS代理
SS默认是针对浏览器的请求进行代理，很多时候我们也希望Console也能够翻墙，例如Golang的`dep ensure -update`命令，因此如果想要在终端(命令行)使用SS进行代理就得做一些配置。

1. 安装privoxy  
   `brew install privoxy`
   
2. 配置privoxy  
   `vim /usr/local/etc/privoxy/config`  
   ```
   confdir /usr/local/etc/privoxy
   logdir /usr/local/var/log/privoxy
   actionsfile match-all.action # Actions that are applied to all sites and maybe overruled later on.
   actionsfile default.action   # Main actions file
   actionsfile user.action      # User customizations
   filterfile default.filter
   filterfile user.filter      # User customizations
   logfile logfile
   listen-address 0.0.0.0:8118  # 监听任意IP地址的8118端口
   toggle  1
   enable-remote-toggle  0
   enable-remote-http-toggle  0
   enable-edit-actions 0
   enforce-blocks 0
   buffer-limit 4096
   enable-proxy-authentication-forwarding 0
   forwarded-connect-retries  0
   accept-intercepted-requests 0
   allow-cgi-request-crunching 0
   split-large-forms 0
   keep-alive-timeout 5
   tolerate-pipelining 1
   socket-timeout 300

   forward-socks5 /  127.0.0.1:1080 .   # 代表所有的URL都转发到socks5代理，1080为默认socks端口 需要和SS配置的端口保持一致
   forward     192.168.*.*/     .
   forward     172.16.*.*/     .
   forward     172.17.*.*/     .
   forward     172.18.*.*/     .
   forward     172.19.*.*/     .
   forward     172.20.*.*/     .
   forward     172.21.*.*/     .
   forward     172.22.*.*/     .
   forward     172.23.*.*/     .
   forward     172.24.*.*/     .
   forward     172.25.*.*/     .
   forward     172.26.*.*/     .
   forward     172.27.*.*/     .
   forward     172.28.*.*/     .
   forward     172.29.*.*/     .
   forward     172.30.*.*/     .
   forward     172.31.*.*/     .
   forward     10.*.*.*/     .
   forward     127.*.*.*/     .
   forward     localhost/     .
   forward     .local.com/     .
   ```

3. 启动privoxy  
   `sudo /usr/local/sbin/privoxy /usr/local/etc/privoxy/config`
   
4. 查看是否启动成功  
   `netstat -na | grep 8118`  
   有如下信息就表示启动成功了  
   `tcp4       0      0  *.8118                 *.*                    LISTEN`
   
5. 终端开启代理 (只针对当前窗口有效，可以通过设置~/.bashrc来保证新窗口生效)    
   `export http_proxy='http://localhost:8118'`  
   `export https_proxy='https://localhost:8118'`
   
6. 关闭代理   
   `unset http_proxy`
   `unset https_proxy`
      
7. 相关问题
   + `Failed to connect to localhost port 8118: Connection refused`: 确认下 privoxy 是不是没有开启

