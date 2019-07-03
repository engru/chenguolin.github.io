---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

1. mysql -hhostname -uuser -ppsword -e “mysql_cmd"
2. mysql -hhostname -uuser -ppsword << EOF
    mysql_cmd
    EOF
3. 如下简单例子：
    #!/bin/bash
    mysql -hservicedb-online -uroot -proot123 -e "use test;select * from tests;"  
    #方法1实例
    mysql -hservicedb-online -uroot -proot123 << EOF   #方法2实例
    use test;
    select * tests;
    EOF
