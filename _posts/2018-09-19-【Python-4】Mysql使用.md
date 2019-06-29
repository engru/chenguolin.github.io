---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

1. 查询文档: [http://www.runoob.com/python/python-mysql.html](http://www.runoob.com/python/python-mysql.html)
2. db = MySQLdb.connect(host="10.99.84.66",port=3309,  
   user="ro_shenma_youhui",passwd="CqaaFhCyecAKMewz",db="shenma_youhui", charset='utf8')  
   cursor = db.cursor()  
   cursor.execute("SELECT link, title from feed_kuaibao where id = %s" % id)  
   data = cursor.fetchone()  
   db.close()  
3. 常见问题
    * select出的字段包含有中文导致乱码
      解决方案: MySQLdb connect的时候指定charset='utf-8'
    * insert中文出现: 'latin-1' codec can't encode characters in position 0-3: ordinal not in range(256)
      解决方案: MySQLdb connect的时候指定charset='utf-8'
