---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. svn命令
1. svn删除远程仓库但是保持本地仓库: `$ svn del spam_host_recover/build spam_host_recover/result --keep-local` 
2. svn创建branch  
   + `$ svn cp http://svn.xxx.com/svn/apsara/golang/common/trunk/ http://svn.xxx.com/svn/apsara/golang/common/branches/chenguolin_20141024  -m "mkdir branch"`
   + `$ svn switch http://svn.xxx.com/svn/apsara/golang/common/branches/chenguolin_20141024`  
   + `$ svn info`  //查看是否切换到branch  
3. svn branch合并
   + 先切换到trunk
   + `$ svn merge --reintegrate http://svn.xxx.com/svn/apsara/golang/common/branches/chenguolin_20141024`
   + 查看diff是否merge成功
4. svn查看branch和trunk的diff
   + `$ svn diff --old http://svn.xxx.com/svn/apsara/golang/common/branches/chenguolin_20141024 --new http://svn.xxx.com/svn/apsara/golang/common/trunk/`
5. svn revert
   + revert文件: `$ svn revert file`
   + revert某个文件夹: `$ svn revert -R dir`
6. svn恢复某个文件夹: `$ svn update dir`   
7. svn两个branch merge，例如yyy/branch merge到 xxx/branch
   + `$ svn switch xxx/branch`  //先切换到branch
   + `$ svn merge --reintegrate yyy/brahch .`
   + `$ svn diff` //查看diff

# 二. awk和sed命令
1. sed大写字母转换成小写字母: `$ sed 's/[A-Z]/\l&/g' file`
2. mac某个目录下文件统一替换: sed -i "" "s/xxxxx/yyyy/g" `grep '"httpserver/' -rl .`  //mac sed命令需要加-i参数

# 三. find命令
1. 删除某个目录下所有.svn文件夹: `$ find . -name *svn | xargs rm -rf`
2. 把某个文件夹下对应所有.svn移动到另一个对应文件夹下: `$ find . -name .svn -print0 | xargs -0 -I {} mv '{}' '../browse_log/{}'`
3. 删除最近8天未更新的文件: `$ find . -mtime +8 -name "*.log" -exec rm -rf {} \`
4. 列出当前目录非空文件: `$ find -maxdepth 1 -type f ! -empty -exec ls -lh {} +`
5. 列出某个目录非空子目录: `$ find /var/lib/docker/devicemapper/mnt/ -maxdepth 2 -type f ! -empty -exec ls -lh {} +`
6. 查找txt和pdf文件: `$ find . \( -name "*.txt" -o -name "*.pdf" \) -print` 等价于 `$ find . -regex  ".*\(\.txt|\.pdf\)$"`
7. 查找所有非txt文本: `$ find . ! -name "*.txt" -print`
8. 列出当前目录的文件: `$ find . -maxdepth 1 -type f`
9. 最近7天内被访问过的所有文件: `$ find . -atime -7 -type f -print`
10. 查找7天前被访问过的所有文件: `$ find . -atime +7 type f -print`
11. 查找大于2k的文件: `$ find . -type f -size +2k`

# 四. rpm命令
1. 查询程序是否安装: `$ rpm -q samba`
2. 搜索指定rpm包是否安装: `$ rpm -qa | grep httpd`
3. 搜索rpm包: `$ rpm -ql httpd`
4. 查看rpm包信息: `$ rpm -qpi Linux-1.4-6.i368.rpm`
5. 查看rpm包的文件: `$ rpm -qpf Linux-1.4-6.i368.rpm`
6. 安装新的rpm包: `$ rpm -ivh file.rpm`
7. 升级一个rpm包: `$ rpm -Uvh file.rpm`
8. 卸载一个rpm包: `$ rpm -e file.rpm`
9. 指定路径查看是否安装某些rpm包: `$ rpm -qa --root=/home/admin/chenguolin/workdir/project/_external/ | grep google_url`
10. 查看某些文件是属于哪个rpm包: `$ rpm -qf /usr/local/lib64/libjvm.so`
11. 查看rpm包的默认安装目录: `$ rpm -qpl xxx.rpm`
12. 指定位置安装rpm包: `$ rpm -ivh --relocate /usr/bin=/home/admin/chenguolin xxx.rpm`
13. rpm包解压: `$ rpm2cpio xx.rpm | cpio -div`
14. fpm命令打rpm包: `$ fpm -s dir -t rpm --iteration 1.el6 -v 0.0.1 -n pgproxy --description "paogao proxy" --prefix="/usr/local/" --post-install post_install.sh --post-uninstall post_unins tall.sh --after-upgrade post_upgrade.sh pgproxy`
    * -s: 指定源类型
    * -t: 指定目标类型
    * --iteration: release版本
    * -v: 版本
    * -n: rpm名称
    * --description: 描述
    * --prefix: rpm包默认安装路径 
    * --post-install: rpm安装成功后执行的脚本
    * --post-uninstall: rpm 卸载后执行的脚本
    * —after-upgrade: 更新rpm包后要执行的脚本
    * pgproxy: 要打包的目录

# 五. yum命令
1. 查看哪些rpm包中包含指定文件: `$ yum provides libjvm.so`
2. 搜索系统下的某些rpm包: `$ yum search xxx`
3. 查看安装的rpm包: `$ yum list xxx`

# 六. mysql命令
1. 登录mysql: `$ mysql -A  -h localhost --port port -u myname -p mypass mydb`
2. shell脚本执行mysql命令: `$ /usr/bin/mysql -h 127.0.0.1 -P 3307 -u cgl --password=cgl@2014 --execute="delete from test.browser_log where createtime < NOW() - INTERVAL 345600 SECOND;"`
3. mysqldump导数据: `$ mysqldump -h localhost -P 8002 -uxxxx -p'xxxx' --databases test --tables users --single-transaction > 20180903_tokens.sql`
4. shell脚本执行mysql命令dump数据: `$ mysql -h localhost -P 8002 -u mynamq -p'xxxxx' -D test -e "select uid, phone, amount from users where timestamp >= 1530374400" > data`

# 七. vim使用
1. 强制保存文件: `w !sudo tee %`   // %代指当前文件名, !sudo tee %表示用sudo权限输出到标准输出并且保存到当前文件中。
2. 可以在`.vimrc`里面配置 `command W w !sudo tee % > /dev/null` 使用W来代替强制保存命令

# 八. scp命令
1. 本地拷贝到远程服务器
   + 文件: `$ scp file.dat root@12.13.14.15:/root/chenguolin`
   + 目录: `$ scp -r dir root@12.13.14.15:/root/chenguolin`
2. 远程服务器拷贝到本地
   + 文件: `$ scp root@12.13.14.15:/root/chenguolin/file.dat .`
   + 目录: `$ scp -r root@12.13.14.15:/root/chenguolin/dir .`
3. scp指定私钥文件
    + 本地拷贝到远程: `$ scp -i ssh-key.pem file.dat root@12.13.14.15:/root/chenguolin`
    + 远程拷贝到本地: `$ scp -i ssh-key.pem root@12.13.14.15:/root/chenguolin/file.dat .`

# 九. [git命令](https://chenguolin.github.io/2017/09/02/Git-2-Git-%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4/)
1. git创建branch: `$ git branch xxx`
2. git显示branch: `$ git branch`
3. git分支push: `$ git push -u origin branch`
4. git master／branch切换
   + branch切换到master: `$ git checkout master`
   + master切换到branch: `$ git checkout branch`
5. git删除本地分支: `$ git branch -D branch`
6. git撤销本地commit: `$ git reset HEAD~1`
7. git checkout远程分支: `$ git checkout --track origin/2017-11-14-optimize-hive-query-cgl`
8. git 删除远程分支: `$ git push origin -d branch_name`
9. git 打tag: `$ git tag v1.4.0`
10. git 查看所有tag: `$ git tag`
11. git 推送本地tag: `$ git push origin --tag xxxx`
12. git checkout 某个tag: `git checkout tag_name`
13. git 删除某个tag
    + 本地tag: `$ git tag -d xxxx`
    + 远程tag: `$ git push origin :refs/tags/xxxx`

# 十. 打包/压缩
1. 创建tar.gz文件: `$ tar -cvzf xxx.tar.gz /xxx/yyy/zzz`
2. 解压tar.gz文件: `$ tar -xvzf xxx.tar.gz`

# 十. 进程相关
1. pgrep查询进程
   + 按名称查找进程: `$ pgrep -l httpd`
   + 按父进程ID查找子进程: `$ pgrep -P {PID}`
2. 查看进程是否因为OOM被kill
   + `$ grep oom /var/log/messages`
   + `$ grep oom /var/log/kern`
3. 查看最耗CPU的进程: `$ top -c` （输入大写P）
4. 查看进程运行的全命令
   + `$ ps -fp {PID}`
   + `$ cat /proc/<pid>/cmdline`
5. 常用变量
   + `$$` 当前进程ID
   + `$PPID` 当前进程父进程ID
6. pidof根据关键字查找所有相关进程
   + `$ pidof docker`
7. kill相关命令
   + 发送`SIGHUP`信号用于通知进程重新加载配置文件: `$ kill -1 {PID}`
   + 发送`SIGTERM`信号用于杀死进程: `$ kill -15 {PID}`
   + 发送`SIGKILL`信号强杀进程: `$ kill -9 {PID}`
8. pkill和killall相关命令
   + 根据名称杀死进程: `$ pkill httpd`
   + 根据名称杀死所有进程: `$ killall httpd`
9. lsof命令 (list opened files)
   + 查看端口占用的进程状态: `$ lsof -i:3306`
   + 查看用户的进程所打开的文件: `$ lsof -u {username}`
   + 查询指定的进程ID打开的文件: `$ lsof -p {pid}`
   + 查询指定目录下被进程开启的文件: `$ lsof +D {dir}`  (+D 递归目录)
   + 查找那些被显式删除但文件描述符还未释放的文件: `$ lsof | grep deleted`

# 十一. 日志查看
1. 查看磁盘设备相关信息日志: `$ dmesg`
2. 查看系统日志: `$ tail -f /var/log/messages`
3. 查看cron日志: `$ tail -f /var/log/cron`

# 十二. 磁盘管理
1. 查看当前支持文件系统类型: `$ cat /proc/filesystems`
2. mount相关命令
   + 挂载目录，设置为ro: `$ mount -t ext2 -o ro /dev/hdb1 /home/project42`
   + 挂载目录，设置为noexec: `$ mount -t ext2 -o noexec /dev/hdb1 /home/project42`
   + 列出当前所有挂载点: `$ mount`
3. umount相关命令
   + 卸载某个挂载点: `$ mount -l /home/project42`
4. 查看磁盘空间使用情况
   + 查看磁盘使用率: `$ df -h`  (-h: human缩写以易读的方式显示结果（即带单位: 比如M/G，如果不加这个参数，显示的数字以B为单位)
   + 查看/home目录使用空间: `$ du -sh /home`  (-h 人性化显示; -s 递归整个目录的大小)
   + 查看当前目录下所有文件大小: `$ du . | sort -k1rn`
   + 查看当前目录下一级子目录大小: `$ du . -c --max-depth=1 | sort -k1rn`
5. docker容器挂载相关
   + 查看当前容器所有挂载点: `$ mount` 或者 `$ cat /proc/mounts`
   + `/proc/mounts` 实际上是软链接到 `/proc/self/mount`
   + 查看详细的挂载点信息: `$ cat /proc/self/mountinfo`
   + 查看挂载点的状态: `$ cat /proc/self/mountstats`
   + 查看容器被哪些进程挂载: `$ grep -l {container-id} /proc/*/mountinfo`
   
# 十三. 网络
1. 查看服务网络是否能通
   + telnet命令: `$ telnet 10.11.12.13 8888`
   + tcpping命令: `$ tcpping 10.11.12.13 port`
2. 查看某个端口是否被占用
   + `$ lsof -i:1234`
   + `$ netstat -apn | grep 1234`
3. netstat命令
   + 列出所有端口 (包括监听和未监听的): `$ netstat -a`
   + 列出所有TCP端口: `$ netstat -at`
   + 列出所有有监听的服务状态: `$ netstat -l`
4. 网络路由
   + 查看路由状态: `$ route -n`
   + 发送ping包到地址IP: `$ ping {IP}`
   + 探测前往地址IP的路由路径: `$ traceroute {IP}`
5. 网络下载: `wget` 或 `url`
   + `–limit-rate`: 下载限速
   + `-o`: 指定日志文件
   + `-c`: 断点续传
6. curl命令
   + GET请求: `curl -H 'Content-Type:application/json' -H "Cookie:xxxx"  "http://xxx/..."`
   + POST请求: `curl -H 'Content-Type:application/json' -H "Cookie:xxxx" -d '{....}' "http://xxx/..."`
   
# 十四. 文件
1. 查看两个文件间的差别: `$diff file1 file2`
2. 计算2个文件交集和差集
    + 求两文件交集: `$ grep -f file1 file2`
    + 求两文件差集: `$ grep -vf file1 file2`
3. egrep查询文件内容: `$ egrep '03.1\/CO\/AE' test.log`
4. 创建软链接/硬链接
    + 创建软链接: `$ ln -s {src} {dst}`  (src表示源路径，dst表示目标路径)
    + 创建硬链接: `$ ln {src} {dst}`     (不支持创建目录硬链接，src表示源文件路径，dst表示目标文件路径)
5. 两个文件拼接: `$ paste {file1} {file2}`

# 十五. 权限
1. 改变拥有者
    + 改变文件: `$ chown cgl:cgl {file}`
    + 改变目录: `$ chown -R cgl:cgl {dir}` (-R 递归子目录修改)
2. 改变文件读、写、执行等属性
    + 增加脚本可执行权限: `$ chmod a+x myscript`

# 十六. 环境变量
1. 系统全局环境变量配置文件: `/etc/profile` 和 `/etc/bashrc`
2. 用户目录下的私有环境变量配置文件: `~/.profile` 和 `~/.bash_profile` 和 `~/.bashrc` 等
3. 打开shell进程，读取环境设置脚本分为以下3步
    + 先读入全局环境变量设置文件 `/etc/profile`，然后根据其内容读取额外的配置文件如 `/etc/profile.d`和 `/etc/inputrc`
    + 再读取当前登录用户家目录下的文件 `~/.bash_profile`，其次读取 `~/.bash_login`，最后读取 `~/.profile`
    + 最后读取当前登录用户家目录下的文件 `~/.bashrc`

# 十七. 系统管理
1. 查看系统版本: `$ uname -a`
2. 查看硬件架构: `$ arch`
3. 查看CPU使用情况: `$sar -u 5 10`  (每5秒采集一个点，共采集10个点)
4. 查看CPU核的个数: `$ cat /proc/cpuinfo | grep processor | wc -l`
5. 查看内存信息: `$cat /proc/meminfo`
6. 查看系统级别限制: `$ ulimit -a`
7. 设置不限制生成`core`文件大小: `$ ulimit –c unlimited`
8. 查看系统可用内存: `$ free -h`
9. 查看CPU信息和型号: `$ cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c`

# 十八. 其它
1. 运行C++ 可执行文件
   + `$ /bin/env LD_LIBRARY_PATH=build/release64/packages/lib/ url_tool`  //LD_LIBRARY_PATH指定依赖的so路径
2. 域名解析: `$ dig www.baidu.com`
3. Python命令格式化json数据: `$ python -mjson.tool json.dat`
4. 查看kafka server版本: `$ sudo find / -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*’`
5. Mac进入自带的docker-for-desktop节点: `$ screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty`
6. 清理所有的screen窗口: `$ pkill SCREEN`
7. Shell计算base64编码
    + Encode: `$ echo "cgl" | base64`
    + Decode: `$ echo 'MWYyZDFlMmU2N2Rm' | base64 —decode`

# 十九. 差异
1. Ubuntu操作系统没有 `dig` 工具，需要安装 `dnsutils`，命令为 `apt-get install dnsutils`




