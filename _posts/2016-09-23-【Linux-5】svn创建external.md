---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

1. 显示当前svn目录下所有external
    svn proplist
2. 删除当前目录下指定的external
    svn propdel PROPNAME
    例如
    svn propdel svn:externals
3. 编辑当前目录external
    svn propedit svn:externals .
 4. 查看一下是否正确
     svn proplist
     svn propget svn:externals
5. 提交并update
    svn ci -m "testeste"
    svn up
