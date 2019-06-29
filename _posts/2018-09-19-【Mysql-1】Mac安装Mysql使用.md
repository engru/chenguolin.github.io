---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Mysql
---

# 一. 安装Mysql
1. 安装命令  
   `brew install mysql@5.7`  
   注意Mysql 8.0版本和5.7版本差别很大，5.7很多权限相关的命令都不能在8.0版本上使用
     
2. 启动Mysql  
   `mysql.server start`

3. 登录  
   `mysql -h 127.0.0.1 -P 3306 -u root -p`  
   首次登录是没有密码的，直接回车即可

4. 重置密码  
   `SET PASSWORD = PASSWORD('root');`

5. 使用密码登录  
   `mysql -h 127.0.0.1 -P 3306 -u root -p'root'`

# 二. 卸载Mysql
  完全卸载Mysql可以按照以下几个步骤来实现，什么情况下需要完全卸载Mysql呢，有些时候本地Mysql环境被破坏重装之后一直无法正常启动，就可以采用完全删除再重装的方式。  
  参考: https://gist.github.com/vitorbritto/0555879fe4414d18569d

1. 备份数据  
   `使用mysqldump`

2. 停止Mysql服务  
   `mysql.server stop`

3. 使用HomeBrew卸载Mysql  
   `brew remove mysql`  
   `brew cleanup`

4. 删除相关目录文件  
   `sudo rm /usr/local/mysql`
   `sudo rm -rf /usr/local/var/mysql`  
   `sudo rm -rf /usr/local/mysql*`  
   `sudo rm ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist`  
   `sudo rm -rf /Library/StartupItems/MySQLCOM`  
   `sudo rm -rf /Library/PreferencePanes/My*`
   
