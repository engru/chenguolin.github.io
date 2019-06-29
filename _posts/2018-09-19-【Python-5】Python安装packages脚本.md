---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

```
#!/bin/sh

## 依赖包安装目录
INSTALL_DIR=$(pwd)/install_dir

if [ ! -d $INSTALL_DIR ]; then
      mkdir $INSTALL_DIR
fi

cd $INSTALL_DIR

## python版本
PYTHON=/home/admin/.pythonbrew/pythons/Python-2.7.[0-9]/bin/python

## 安装tornado
wget https://pypi.python.org/packages/source/t/tornado/tornado-4.2.tar.gz --no-check-certificate
tar -xzf tornado-4.2.tar.gz
cd tornado-4.2
$PYTHON setup.py build
$PYTHON setup.py install
cd ..

## 安装futures
wget https://pypi.python.org/packages/source/f/futures/futures-3.0.3.tar.gz#md5=32171f72af7e80c266310794adc4db46 --no-check-certificate
tar -xzf futures-3.0.3.tar.gz
cd futures-3.0.3
$PYTHON setup.py install
cd ..

## 安装requests
wget https://pypi.python.org/packages/source/r/requests/requests-2.9.1.tar.gz#md5=0b7f480d19012ec52bab78292efd976d --no-check-certificate
tar -xzf requests-2.9.1.tar.gz
cd requests-2.9.1
$PYTHON setup.py install
cd ..

## 安装selenium
wget https://pypi.python.org/packages/source/s/selenium/selenium-2.50.1.tar.gz#md5=c2dc97dab1bf499a57a5b665814c2e75 --no-check-certificate
tar -xzf selenium-2.50.1.tar.gz
cd selenium-2.50.1
$PYTHON setup.py install
cd ..

##安装Pillow，代替按照PIL
wget https://pypi.python.org/packages/source/P/Pillow/Pillow-3.0.0.tar.gz#md5=fc8ac44e93da09678eac7e30c9b7377d --no-check-certificate
tar -xzf Pillow-3.0.0.tar.gz
cd Pillow-3.0.0
$PYTHON setup.py install
cd ..

## 安装six
wget https://pypi.python.org/packages/source/s/six/six-1.10.0.tar.gz#md5=34eed507548117b2ab523ab14b2f8b55 --no-check-certificate
tar -xzf six-1.10.0.tar.gz
cd six-1.10.0
$PYTHON setup.py install
cd ..

## 删除安装目录
rm $INSTALL_DIR -rf
```
