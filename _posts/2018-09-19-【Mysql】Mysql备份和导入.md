---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Mysql
---

1. 备份
    mysqldump --default-character-set=utf8 --skip-lock-tables -hxx.xx.xx.xx -Pport -uUsername -pPassword db table > name.sql

2. 导入
     mysql --default-character-set=utf8 -h xx.xx.xx.xx --port port -u username -f -pPassword db < name.sql

3. 中文乱码问题
    * 导入和导出 都使用 --default-character-set=utf8
    * 跳过锁表: --skip-lock-tables
