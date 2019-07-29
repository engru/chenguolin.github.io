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

1. `/etc/cron.allow`: 允许配置cron job的用户列表，文件允许不存在
2. `/etc/cron.deny`: 不允许配置cron job的用户列表，文件允许不存在
3. `/etc/crontab`: 系统级别cron job配置文件，不建议用户直接修改该文件
4. `/etc/cron.hourly/`: `小时`级别调度任务配置目录，目录下脚本可以被系统自动调用
5. `/etc/cron.daily/`: `天`级别调度任务配置目录，目录下脚本可以被系统自动调用
6. `/etc/cron.weekly/`: `周`级别调度任务配置目录，目录下脚本可以被系统自动调用
7. `/etc/cron.monthly/`: `月`级别调度任务配置目录，目录下脚本可以被系统自动调用
8. `/etc/cron.d/`: 非小时、天、周、月级别的调度任务配置目录，需要执行命令 `crontab -u username /etc/cron.d/xxxx` 载入到用户crontab文件
9. `/var/spool/cron`: 存放所有用户的crontab文件目录

上面提到的几个文件，有几个需要注意的点，总结如下
1. `/etc/cron.allow` 和 `/etc/cron.deny` 正常情况下不需要这2个文件，保证所有人都可以配置cron job
2. `/etc/crontab` 不建议用户直接修改这个文件，每个用户独立编辑各自crontab文件
3. `/etc/cron.hourly/`、`/etc/cron.daily/`、`/etc/cron.weekly/`、`/etc/cron.monthly/` 这4个目录下的脚本需要满足以下条件，才能够自动被调度执行
   + 文件名只能包含 `[a-zA-Z0-9_-]` 这些字符，同时不能使用 `.sh` 结尾
   + 脚本必须要有 `可执行` 权限
   + 文件第一行必须为 `#!/bin/bash`
4. crontab任务运行的最小时间间隔是`1分钟`，也就是最细粒度只能支持分钟级别任务定时调度，不支持秒级任务定时调度

# 二. crontab 命令
1. crontab 服务
   + 启动: `$ service crond start`
   + 停止: `$ service crond stop`
   + 状态: `$ service crond status`
   + 重启: `$ service crond restart`
2. crontab 常用命令
   + cron job文件载入到某个用户的crontab文件: `$ crontab -u {username} {file}` 
   + 显示某个用户的crontab文件内容: `$ crontab -u {username} -l`  
   + 编辑某个用户的crontab文件内容: `$ crontab -u {username} -e`  
   + 删除某个用户的crontab文件内容: `$ crontab -u {username} -ir`  
   
   `如果没有通过 -u 指定用户，默认使用当前用户，某个用户的crontab文件在/var/spool/cron目录下`
3. run-parts 可以用于执行 `/etc/cron.hourly/`等目录下脚本 （可以用来debug自己写的调度脚本是否能成功被执行）
   + `run-parts /etc/cron.hourly`
   + `run-parts /etc/cron.daily`
   + `run-parts /etc/cron.weekly`
   + `run-parts /etc/cron.monthly`

# 三. crontab 使用
正常情况下如果开发或运维有定时任务需求都可以通过 `crontab` 来实现，但是配置 `crontab` 需要登录机器这会带来一定的风险。现在有很多开源的 `任务调度` 平台可以用来实现我们这个需求，除此之外 `K8s` 的`Job`和`CronJob`也能够支持任务定时调度。

所以，如果只是简单的需求可以直接使用 `crontab` 配置，但是非常不推荐在生产环境大量使用，因为不仅低效而且风险很大。

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
2. 根据不同定时调度需求，编写调度脚本放在不同目录下
   + 小时级别调度: 放在 `/etc/cron.hourly` 目录
   + 天级别调度: 放在 `/etc/cron.daily` 目录
   + 周级别调度: 放在 `/etc/cron.weekly` 目录
   + 月级别调度: 放在 `/etc/cron.monthly` 目录
3. 编写cron job文件，使用 `$ crontab -u {username} {file}` 载入到用户crontab文件中
   + 拷贝cron job文件到 `/etc/cron.d` 目录下  (该目录下所有文件，必须满足crontab文件内容格式)
   + 执行命令载入到用户crontab文件: `$ crontab -u {username} /etc/cron.d/xxxx`
   + 查看某个用户的crontab文件内容: `$ crontab -u {username} -l`
4. Dockerfile中使用
   + 安装cron: `apt-get install cron` 或 `yum install cron`  （cron会开机自启动）
   + COPY命令拷贝cron job文件到 /etc/cron.d目录 `COPY xxxx /etc/cron.d/xxx`
   + RUN命令载入到用户crontab文件: `RUN crontab /etc/cron.d/xxx`   （不指定-u 默认使用当前用户）

# 四. 最佳实践
通过日常的一些实践，配置crontab遵循以下几个原则，保证稳定和可维护

1. crontab配置定时调度执行某个shell脚本，执行命令封装在shell脚本内  
   `* * * * * cd /home/cgl/tmp && sh run.sh`  
   `3,15 * * * * cd /home/cgl/tmp2 && sh run2.sh`
2. 用户具体要执行的命令封装在shell内
   ```
   #!/bin/sh
   touch `date +%s`.dat
   ...
   ```
3. 每一条cron job都要有注释，写明是谁配置和job对应的功能
   
# 五. crontab 举例
1. 每分钟执行: `* * * * * command`
2. 每5分钟执行: `*/5 * * * * command`
3. 每个小时第10分执行: `10 * * * * command`
4. 每个小时第25、35、45分执行: `25,35,45 * * * * command`
5. 每天8-11点的第25分钟执行ls命令: `25 8-11 * * * command`
6. 每隔两天的上午8点到11点的第3和第15分钟执行: `3,15 8-11 */2  *  * command`
7. 每周一上午8点到11点的第3和第15分钟执行: `3,15 8-11 * * 1 command`

# 六. 注意事项
1. 新创建的cron job，不会马上执行至少要过`2分钟`才执行，如果重启`crond`则马上生效
2. 当crontab失效时，可以尝试 `$ service crond restart` 解决问题，或者查看日志确认某个job有没有执行报错 `$ tail -f /var/log/cron`
3. `crontab -u {username} -r` 命令会把 `var/spool/cron` 目录对应用户crontab文件删除，删除之后该用户的所有cron job全部失效
4. crontab中`%`是有特殊含义的表示换行的意思，如果要用的话必须进行 `\` 进行转义，例如 `date +%s` 必须使用 `date +\%s`代替。
5. 将每条任务进行重定向处理非常重要，如果任务输出的日志很多日积月累可能会影响正常的系统运行，可以考虑重定向到 `/dev/null`  
   例如`0 */3 * * * sh run.sh >/dev/null 2>&1`，`/dev/null 2>&1`表示先将`stdout`重定向到`/dev/null`，然后将`stderr`重定向到`stdout`，由于`stdout`已经重定向到了`/dev/null`，因此`stderr`也会重定向到`/dev/null`。
6. 如果我们创建了一个cron job，但是这个任务却无法自动执行，而手动执行这个任务却没有问题，这种情况一般是由于在crontab文件中没有配置环境变量引起的。
   所以我们要保证在shelll脚本中提供所有必要的路径和环境变量，除了一些自动设置的全局变量，脚本中涉及文件路径时写全局路径。
