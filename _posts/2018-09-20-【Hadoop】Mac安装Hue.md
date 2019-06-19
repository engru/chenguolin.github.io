---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Hadoop
---

1.  clone Hue Repository
     git clone [https://github.com/cloudera/hue.git](https://github.com/cloudera/hue.git)

2.  install pre-requisties
    * ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)”;
    * brew doctor
    * brew update
    * brew install maven
    * brew install mysql

3.  install openssl
    * brew install openssl
    * export LDFLAGS=-L/usr/local/opt/openssl/lib && export CPPFLAGS=-I/usr/local/opt/openssl/include

4.  compile
    * make apps

5.  config
    * sudo vim /etc/hosts
    * 添加172.16.156.130 quickstart.cloudera 

6.  run 
    * ./build/env/bin/hue runserver
