---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Git
---

# 一. 前言
Gitlab是一个基于Git实现的在线代码仓库软件，你可以用Gitlab自己搭建一个类似于Github一样的系统，一般用于在企业、学校等内部网络搭建Git服务。

Gitlab官方提供了很多安装的方式，具体可以参考[gitlab install](https://about.gitlab.com/install/)

这篇文章介绍用docker的方式安装Gitlab，因为docker部署起来非常的方便，下面进入正题~

关于docker是什么可以自行Google了解，docker的安装可以参考[docker install](https://docs.docker.com/install/overview/)

# 二. Gitlab安装
1. 下载gitlab-ce镜像，镜像比较大需要等待一段时间  
   docker pull gitlab/gitlab-ce:latest
2. 数据挂载，为了保证容器内的数据能够持久化存储我们需要在本地创建目录，然后容器运行的时候映射到当前本地目录  
   mkdir -p ~/data/gitlab/config  
   mkdir -p ~/data/gitlab/data  
   mkdir -p ~/data/gitlab/logs  
3. 运行gitlab镜像  
   docker run \  
      --publish 443:443 \  
      --publish 80:80 \  
      --publish 22:22 \  
      --volume ~/data/gitlab/config:/etc/gitlab \  
      --volume ~/data/gitlab/logs:/var/log/gitlab \  
      --volume ~/data/gitlab/data:/var/opt/gitlab \  
      --name gitlab \  
      gitlab/gitlab-ce:latest  
   本地端口443、80、和22直接转发到容器相同的端口上
4. 配置本地域名访问  
   sudo vi /etc/hosts  在默认添加一行gitlab.local.com

# 三. Gitlab配置
1. 重启Gitlab可以使用  
   `docker restart b2e5fc3b4e38`
2. 配置IP或域名  
   `vi ~/gitlab/config/gitlab.rb`  
   修改：external_url '服务器IP/已备案的域名'  
   示例：external_url 'http://gitlab.local.com/'
3. 配置邮箱服务  
   `vi ~/gitlab/config/gitlab.rb`  
   gitlab_rails[‘smtp_enable’] = true   
   gitlab_rails[‘smtp_address’] = "smtp.163.com"  
   gitlab_rails[‘smtp_port’] = 25  
   gitlab_rails[‘smtp_user_name’] = "xxx@163.com"   
   gitlab_rails[‘smtp_password’] = "xxx"  
   gitlab_rails[‘smtp_domain’] = "163.com"  
   gitlab_rails[‘smtp_authentication’] = :login  
   gitlab_rails[‘smtp_enable_starttls_auto’] = true  
   gitlab_rails[‘gitlab_email_from’] = "xxx@163.com"   
   user[“git_user_email”] = "xxx@163.com"  
4. 重新加载配置  
   进入容器 `docker exec -it gitlab bash`  
   加载配置 `gitlab-ctl reconfigure`

# 四. Gitlab使用
1. 访问Gitlab，容器启动成功后访问 http://gitlab.local.com 第一次登录Gitlab需要修改管理员密码，管理员账号默认为root  
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-modify-root-password.png?raw=true)
2. 登录Gitlab，设置完管理员密码后，使用root和密码即可登录
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-login-page.png?raw=true)
3. 登录成功后，root用户就可以开始使用Gitlab
    * 创建一个新的Group
    * 创建一个新的Project
    * 个人信息设置
    ...
4. 管理员有专门的管理员空间可以进行各种统计分析查看、监控和相关的设置
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-admin.png?raw=true)
  
# 五. 普通用户
1. 注册新用户，访问http://gitlab.local.com/users/sign_in 注册新用户
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-register.png?raw=true)
2. 用户登录后，可以进行设置更改
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-user-setting.png?raw=true)
3. 配置SSH keys
   拷贝本地~/.ssh/id_rsa.pub文件内容到Gitlab上
4. 创建group、project并开始使用Gitlab

# 六. 启动命令
1. 启动gitlab容器  
   `docker run -d -it --publish 443:443 --publish 80:80 --publish 22:22 --volume ~/data/gitlab/config:/etc/gitlab --volume ~/data/gitlab/logs:/var/log/gitlab --volume ~/data/gitlab/data:/var/opt/gitlab --name gitlab gitlab/gitlab-ce:latest`
2. 启动runner容器  
   `docker run -d --name gitlab-runner --restart always --link gitlab:gitlab -v /var/run/docker.sock:/var/run/docker.sock -v ~/data/gitlab/runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest`
   
