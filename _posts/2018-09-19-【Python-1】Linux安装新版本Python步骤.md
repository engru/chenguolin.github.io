---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

# 一. 安装依赖库
yum -y install python-devel openssl openssl-devel gcc sqlite sqlite-devel mysql-devel libxml2-devel libxslt-devel tkinter tk-devel

#下载Python
1. mkdir  /tmp/python
2. cd /tmp/python
3. wget [https://www.python.org/ftp/python/2.7.13/Python-2.7.13.tgz](https://www.python.org/ftp/python/2.7.13/Python-2.7.13.tgz)

# 二. 解压Python
1. cd /tmp/python
2. tar -zxf Python-2.7.13.tgz

# 三. 编译安装
1. cd Python-2.7.13
2. ./configure
3. make
4. make install

# 四. 给新版本Python创建软连接
1. ln -s /usr/local/python2.7/lib/libpython2.[7.so](http://7.so)  /usr/lib
2. ln -s /usr/local/python2.7/lib/libpython2.7.so.1.0 /usr/lib
3. ln -s /usr/local/python2.7/bin/python2.7 /usr/bin/python
4. ln -s /usr/local/python2.7/lib/libpython2.[7.so](http://7.so) /usr/lib64
5. ln -s /usr/local/python2.7/lib/libpython2.7.so.1.0 /usr/lib64

# 五. 安装pip
1. wget [https://bootstrap.pypa.io/get-pip.py](https://bootstrap.pypa.io/get-pip.py)
2. python [get-pip.py](http://get-pip.py)

# 六. 安装easy-install
1. wget [https://bootstrap.pypa.io/ez_setup.py](https://bootstrap.pypa.io/ez_setup.py)
2. python [ez_setup.py](http://ez_setup.py)

# 七. Python中文编码
Python默认情况使用ASCII编码来读取源码，所以中文就会出现乱码。  
Python解决中文编码只需要在文件开头输入以下内容
```
#!/usr/bin/python
# -*- coding: utf-8 -*-
```


