---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Git
---

# 一. 背景
日常开发过程中大部分都是属于`多人协作`同时开发某个项目，使用最多的是版本控制系统是`Git`，在一些传统的公司可能使用的是`Svn`或其它的版本控制系统。  
但是无论使用哪种版本控制系统，我们都需要涉及到`commit`，我们先看一些commit log。

```
1. fix comment stripping
2. fixing broken links
3. Bit of refactoring
4. Check whether links do exist and throw exception
5. Fix sitemap include (to work on case sensitive linux)
...
```

对于上述这些commit log我们根本无法知道当时提交修改的目的、修改的文件，虽然能从变更的文件查看变更内容，但是还是有些低效。

因此为了能够在使用 `git log` 命令的时候直观的显示每次commit的目的和变更的文件，每次`git commit`需要按照一定的规范。

# 二. 规范
业内用的最多的 `Git Commit Message` 规范是 [AngularJS Git Commit Message Conventions
](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit#heading=h.uyo6cb12dt6w)

commit message 应该按照以下格式，每个commit message由三部分组成 `header`、`body`以及`footer`，同时每个commit message长度应该小于`100`个字符。
```
<type>(<scope>): <subject>
//空行
<body>
//空行
<footer>
```

1. `<type>`: 本次commit类型
    * `feat`: A new feature
    * `fix`: A bug fix
    * `docs`: Documentation only changes
    * `style`: Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
    * `refactor`: A code change that neither fixes a bug or adds a feature
    * `perf`: A code change that improves performance
    * `test`: Adding missing tests
    * `chore`: Changes to the build process or auxiliary tools and libraries such as documentation generation
2. `<scope>`: 本次变更文件的范围，可选字段。可以是某个类名、包名、函数名、文件名等
3. `<subject>`: 本次commit的主题，言简意赅同时使用小写字母，不需要使用`.`结尾
4. `<body>`: 本次commit的详细内容，可以分点描述
5. `<footer>`: 本次commit是否涉及到不兼容变更或者关闭某个issue，可选字段

如果本次commit是revert前一次commit，commit log应该按照以下规范
```
revert(<scope>): revert previous commit

This reverts commit c8c221ee6420a2c2c9bfac7ac18946631b84f314
```

# 三. 举例
## ① Add new feature
```
feat(kafka): add aliyun kafka client

add kafka package, aliyun kafka client for operator kafka instance:
1.support create new topic
2.support list topic
3.support get topic status
```

## ② Fix bug
```
feat(kafka): modify calculate signature

modify kafka/client.go calculateSignature function, add sort url query params

Closes #392
```

## ③ Change Documentation
```
docs(README): update README

update README file, add content for use aliyun kafka 
```

## ④ Not affect the meaning of the code
```
style(http): add couple of missing semi colons

http/http.go add couple of missing semi colons
```

## ⑤ Add unit test
```
test(http): add unit test

http_test.go add TestRequest function
```

## ⑥ Change build script or dockerfile
```
chore(build): update build.sh and dockerfile

update build.sh and dockerfile for build image
1.build.sh: decrease script arguments，modify docker build command
2.dockerfile: add set timezone 2 Asia/Shanghai
```

