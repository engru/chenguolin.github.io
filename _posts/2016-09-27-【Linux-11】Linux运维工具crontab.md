---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. crontab 介绍
`crontab` 可以用来在固定的间隔时间执行指定的系统指令或 shell 脚本，时间间隔的单位可以是`分钟`、`小时`、`日`、`月`、`周`及以上的任意组合，非常适合用于周期性的日志分析或数据备份等工作。

`crontab` 是Linux自带的定时任务管理工具，用户无须安装就可以直接使用，默认会自带以下文件与目录

1. `/etc/cron.allow`: 允许配置crontab的用户列表，文件允许不存在
2. `/etc/cron.deny`: 不允许配置crontab的用户列表，文件允许不存在
3. `/etc/crontab`: 系统级别crontab配置，不建议用户直接修改该文件
4. `/etc/cron.hourly/`: `小时`级别调度任务配置目录，目录下脚本可以被系统自动调用
5. `/etc/cron.daily/`: `天`级别调度任务配置目录，目录下脚本可以被系统自动调用
6. `/etc/cron.weekly/`: `周`级别调度任务配置目录，目录下脚本可以被系统自动调用
7. `/etc/cron.monthly/`: `月`级别调度任务配置目录，目录下脚本可以被系统自动调用
8. `/etc/cron.d/`: 非小时、天、周、月级别的调度任务配置目录，需要执行命令 `crontab -u username /etc/cron.d/xxxx` 载入crontab

上面提到的几个文件，有几个需要注意的点，总结如下
1. `/etc/cron.allow` 和 `/etc/cron.deny` 正常情况下不需要这2个文件，保证所有人都可以配置crontab任务
2. `/etc/crontab` 不建议用户直接修改这个文件
3. `/etc/cron.hourly/`、`/etc/cron.daily/`、`/etc/cron.weekly/`、`/etc/cron.monthly/` 这4个目录下的脚本需要满足以下条件，才能够成功被调度执行
   + 文件名只能包含 `[a-zA-Z0-9_-]` 这些字符，同时不能使用 `.sh` 结尾
   + 脚本必须要有 `可执行` 权限
   + 文件第一行必须为 `#!/bin/bash`
4. crontab任务运行的最小时间间隔是`1分钟`，也就是最细粒度只能支持分钟级别定时调度，不支持秒级定时调度。 

# 二. crontab 命令
1. crontab 服务
   + 启动: `$ service crond start`
   + 停止: `$ service crond stop`
   + 状态: `$ service crond status`
2. crontab常用命令
   + crontab任务列表文件载入crond进程: `$ crontab -u {username} {file}`
   + 显示某个用户的crontab文件内容: `$ crontab -u {username} -l`  (对应`/var/spool/cron`目录中某个用户的crontab文件)
   + 编辑crontab文件内容: `$ crontab -u {username} -e`   (对应`/var/spool/cron`目录中某个用户的crontab文件)
   + 删除crontab文件内容: `$ crontab -u {username} -ir`  (从`/var/spool/cron`目录中删除某个用户的crontab文件)
   
   `如果没有通过 -u 指定用户，默认使用当前用户`
3. run-parts 可以用于执行 `/etc/cron.hourly/`等目录下脚本 （可以用来debug自己写的调度脚本是否能成功被执行）
   + `run-parts /etc/cron.hourly`
   + `run-parts /etc/cron.daily`
   + `run-parts /etc/cron.weekly`
   + `run-parts /etc/cron.monthly`

# 三. crontab 使用
正常情况下如果开发或运维有定时任务需求都可以通过 `crontab` 来实现，但是配置 `crontab` 需要登录机器这会带来一定的风险。现在有很多开源的 `任务调度` 平台可以用来实现我们这个需求，除此之外 `K8s` 的`Job`和`CronJob`也能够支持任务定时调度。

所以，如果只是简单的需求可以直接使用 `crontab`，但是非常不推荐在生产环境大量使用，因为不仅低效而且风险很大。

使用 `crontab` 有以下几种方式  [crontab 定时任务](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/crontab.html)

1. 直接使用 `$ crontab -u {username} -e` 编辑某个用户的crontab文件，文件每一行格式如下
   ```
   minute hour day  month week command
   0-59  0-23  1-31 1-12  0-6  exec command

   minute: 每个小时的第几分钟执行该任务
   hour: 每天的第几个小时执行该任务
   day: 每月的第几天执行该任务
   month: 每年的第几个月执行该任务
   week: 每周的第几天执行该任务，0表示周日
   command: 指定要执行的程序 、脚本或命令
   ```
2. 根据不同需求编写调度脚本
   + 小时级别调度: 放在 `/etc/cron.hourly` 目录
   + 天级别调度: 放在 `/etc/cron.daily` 目录
   + 周级别调度: 放在 `/etc/cron.weekly` 目录
   + 月级别调度: 放在 `/etc/cron.monthly` 目录
3. 编写crontab任务列表文件，使用 `crontab -u {username} {file}` 载入到用户crond进程
   + 拷贝crontab任务列表文件到 `/etc/cron.d` 目录下   (该目录下所有文件，必须满足crontab文件内容格式)
   + 执行命令载入到crond进程: `crontab -u {username} /etc/cron.d/xxxx`
   + 显示某个用户的crontab文件内容: `$ crontab -u {username} -l`
   
最佳实践: 调度和执行分离，否则可能会遇到一些未知问题，导致一直无法被调度
1. crontab只配置定时调度  
   `* * * * * cd /home/cgl/tmp && sh run.sh`  
   `3,15 * * * * cd /home/cgl/tmp2 && sh run2.sh`
2. 具体执行命令封装在 run.sh 等脚本内
   ```
   #!/bin/sh
   touch `date +%s`.dat
   ...
   ```
3. 如果任务没有被调度，可以查看 `/var/log/cron` 日志确认是否有问题
   
# 四. crontab 举例
1. 每分钟执行: `* * * * * command`
2. 每5分钟执行: `*/5 * * * * command`
3. 每个小时第10分执行: `10 * * * * command`
4. 每个小时第25、35、45分执行: `25,35,45 * * * * command`
5. 每天8-11点的第25分钟执行ls命令: `25 8-11 * * * command`
6. 每隔两天的上午8点到11点的第3和第15分钟执行: `3,15 8-11 */2  *  * command`

