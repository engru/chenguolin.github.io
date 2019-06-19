---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Git
---

# 一. 介绍
上一篇文章我们介绍了[Gitlab 安装使用](https://chenguolin.github.io/2018/12/18/Git-Gitlab-%E5%AE%89%E8%A3%85%E4%BD%BF%E7%94%A8/)，这篇文章我们来介绍Gitlab CI和CD使用。  
Gitlab CI指的是**Continuous Integration**持续集成，CD指的是**Continuous Deployment**持续部署。有了Gitlab CI后我们就可以不再需要使用Jenkins了，编译、构建、测试、部署都可以在Gitlab CI Pipeline上实现非常方便。

从8.0版本开始, Gitlab CI是Gitlab默认的功能，所有项目默认都会启用。  
![](https://docs.gitlab.com/ee/ci/img/cicd_pipeline_infograph.png)

使用Gitlab CI/CD流程如图所示

1. 首先是用户把变更的代码提交到Gitlab
2. 代码提交后就进入CI流程，CI主要做编译构建、单元测试、集成测试
3. CI通过后进入CD流程，CD主要做部署包括Pre环境、Beta环境、Release环境

# 二. CI配置
Gitlab提供了CI服务，如果在项目的根目录添加了`.gitlab-ci.yml`文件并且配置了Runner，那么每一次代码提交都会触发CI Pipeline。`.gitlab-ci.yml`文件告诉Runner做什么事情，默认情况下Pipeline有3个stage分别是`build`,`test`, `deploy`，很多情况下我们不一定需要所有的stage。

总而言之，只需要2步就可以完成CI配置

1. 项目根目录添加一个`.gitlab-ci.yml`文件
2. 配置一个Runner

配置完成以后，每次代码提交Runner都会自动运行一个Pipeline，所有的Pipeline在项目的Pipelines页面可以查看。

## ① .gitlab-ci.yml文件
从7.12版本开始Gitlab CI使用`.gitlab-ci.yml`文件做为项目配置，它放在项目根目录下定义了项目如何构建。  
[Configuration of your jobs with .gitlab-ci.yml](https://docs.gitlab.com/ee/ci/yaml/README.html)

```
# gitlab CI Pipeline配置文件
# 默认使用golang镜像
image: golang:latest

stages:
    - test
    - build
    - rpm

before_script:
    - mkdir -p /go/src/gitlab.local.com
    - ln -s `pwd` /go/src/gitlab.local.com/httpserver && cd /go/src/gitlab.local.com/httpserver
after_script:
    - echo "job run successful ~"

# test stage
# job 1 test go vet
job_govet:
    stage: test
    script:
        - bash ./scripts/pipeline/test_stage/ci-govet-check.sh
    tags:
        - dev
# job 2 test go fmt
job_gofmt:
    stage: test
    script:
        - bash ./scripts/pipeline/test_stage/ci-gofmt-check.sh
    tags:
        - dev
# job 3 test go lint
job_golint:
    stage: test
    script:
        - bash ./scripts/pipeline/test_stage/ci-golint-check.sh
    tags:
        - dev
# job 4 test go unit test
job_unit:
    stage: test
    script:
        - bash ./scripts/pipeline/test_stage/ci-gotest-check.sh
    tags:
        - dev

# build stage
job_build:
    stage: build
    only:
        - master
        - beta
        - develop
        - tags
    script:
        - bash ./scripts/pipeline/build_stage/build.sh
    tags:
        - dev

# rpm stage
job_rpm:
    stage: rpm
    # 定制过的centos镜像 安装了fpm和golang
    image: centos7:custom
    only:
        - master
        - beta
        - develop
        - tags
    script:
        - source /etc/profile
        - bash ./scripts/pipeline/rpm_stage/rpm.sh
    artifacts:
        name: "httpserver-${CI_JOB_NAME}-${CI_COMMIT_REF_NAME}-${CI_BUILD_REF}-${CI_BUILD_ID}"
        expire_in: "3 days"
        paths:
            - cmd/api/*.rpm
            - cmd/cron/*.rpm
            - cmd/processor/*.rpm
    tags:
        - dev
```

上面是httpserver项目的`.gitlab-ci.yml`配置文件，Pipeline运行后如下图所示，3个stage共6个job，下面会重点介绍`.gitlab-ci.yml`关键的配置定义。
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-ci-pipeline.png?raw=true)

### image和service
image和service指定可以被使用的docker镜像和服务

```
image: golang:latest

job_rpm:
    stage: test
    image: centos7:custom
    script:
        - bash ./scripts/pipeline/test_stage/ci-govet-check.sh
    tags:
        - dev
```
1. `image: golang:latest` 全局配置，表示默认情况下job docker executor使用`golang:latest`这个镜像
2. `image: centos7:custom` job_rpm job配置，表示job_rpm job docker executor使用`centos7:custom`这个镜像

### before_script和after_script
`before_script`和`after_script`命令可以分为全局配置和job配置。  
`before_script`是job运行之前执行的脚本命令，`after_script`是job运行之后执行的脚本命令。

```
before_script:
  - global before script
after_script:
  - global after script

job1:
  before_script:
    - execute this instead of global before script
  script:
    - my command
  after_script:
    - execute this after my script

job2:
  script:
    - my command
```

1. job1内部配置了`before_script`和`after_script`，因此job1运行的是自定义的脚本命令
2. job2运行的是全局配置的`before_script`和`after_script`脚本命令

### stages
stages用来定义Pipeline有几个stage，有以下几个特点

1. 同一个stage的所有job的会并行执行
2. 只有当上一个stage所有job都执行成功后才会执行下一个stage
3. 如果`.gitlab-ci.yml`文件没有定义`stages`，那默认会有`build`，`test`，`deploy`3个stage

如下所示，我们定义3个stage
```
stages:
  - build
  - test
  - deploy
```
1. 开始的时候`build` stage所有的job会并行执行
2. `build` stage的所有job都执行成功后才触发`test` stage，同理`test` stage的所有job也是并行执行
3. `test` stage的所有job都执行成功后才触发`deploy` stage，同理`deploy` stage的所有job也是并行执行
4. 所有的stage都执行成功后，本次commit会被标记为`passed`
5. 只要有一个job运行失败，本次commit会被标记为`failed`，剩下的stage都不会再执行

### job和stage
每个job都需要配置属于哪个stage，`stage`关键字用来定义当前job属于哪个stage，例如下面这个配置

```
stages:
  - build
  - test
  - deploy

job 1:
  stage: build  # 定义job 1属于build stage
  script: make build dependencies

job 2:
  stage: build  # 定义job 2属于build stage
  script: make build artifacts

job 3:
  stage: test   # 定义job 3属于test stage
  script: make test

job 4:
  stage: deploy # 定义job 4属于deploy stage
  script: make deploy
```

### script
`script`是每个job必须配置的，表示Runner执行的脚本命令

```
job:
  script: "bundle exec rspec"
```

### only和except
1. `only`定义哪些分支或tag会触发job运行
2. `except`定义哪些分支或tag不会触发job运行

下面这个配置表示job 1这个job只有`master`、`beta`、`develop`和`tags`才会触发执行
```
# build stage
job 1:
    stage: build
    only:
      - master
      - beta
      - develop
      - tags
```

### tags
`tags`用来定义当前job使用哪个Runner，配置Runner的时候会要求设置tag。  
使用`tags`有个好处就是不用job可以使用不同的Runner

```
job:
  tags:
    - dev
```

### artifacts
`artifacts`用来定义job执行成功后哪些文件要被归档  
1. name表示归档后文件名称
2. expire_in表示该归档文件保存多久
3. paths表示哪些文件需要被归档

```
job :
    artifacts:
       name: "httpserver-${CI_JOB_NAME}-${CI_COMMIT_REF_NAME}-${CI_BUILD_REF}-${CI_BUILD_ID}"
       expire_in: "3 days"
       paths:
          - cmd/api/*.rpm
          - cmd/cron/*.rpm
          - cmd/processor/*.rpm
```

### variables
`.gitlab-ci.yml`文件允许用户自定义一些变量，这些变量可以是全局配置也可以是job内部配置，如果是job内部配置变量则会覆盖全局配置。 
```
variables:
  DATABASE_URL: "postgres://postgres@postgres/my_database"
  
job:
  variables:
    DATABASE_URL: "postgres://postgres@postgres/my_database2"
```

另外Gitlab CI有一些内置的变量[GitLab CI/CD Variables](https://docs.gitlab.com/ee/ci/variables/README.html#variables)，重要的变量有以下几个

1. CI_JOB_ID: 当前job的id
2. CI_JOB_NAME: 当前job的名称
3. CI_PIPELINE_ID: 当前pipeline的id
4. CI_PROJECT_ID: 当前项目id
5. CI_PROJECT_NAME: 当前项目名称

## ② Runner配置
Gitlab Runner是一个开源项目用于运行Gitlab CI job并把结果发回Gitlab，它与Gitlab一起使用提供持续集成的服务。  
Gitlab Runner分为两种，Shared runners和Specific runners。Specific runners只能被指定的项目使用，Shared runners则可以运行所有开启Allow shared runners选项的项目。

### Install Runner
安装Runner有很多种方式[Install GitLab Runner](https://docs.gitlab.com/runner/install/index.html)，今天我们主要介绍使用Docker方式安装Runner。  

1. 安装Docker  
   `curl -sSL https://get.docker.com/ | sh`
2. 下载gitlab-runner镜像    
   `docker pull gitlab/gitlab-runner:latest`
3. 创建本地目录用于持久化数据     
   `mkdir -p ~/data/gitlab/runner/config`
4. 运行runner  
   `docker run -d --name gitlab-runner --restart always --link gitlab:gitlab -v /var/run/docker.sock:/var/run/docker.sock -v ~/data/gitlab/runner/config:/etc/gitlab-runner gitlab/gitlab-runner:latest`  
   1). --link gitlab:gitlab 为了保证runner容器能够访问gitlab的容器需要设置docker容器网络连接方式  
   2). -v /var/run/docker.sock:/var/run/docker.sock 指向当前物理机的docker daemon，否则会报`Can't connect to docker daemon ...`这个错误  
   3). -v ~/data/gitlab/runner/config:/etc/gitlab-runner 把runner配置指向本地目录方便修改直接修改配置
   
### Register Runner
Runner下载完成之后还需要进行注册绑定到Gitlab上[Registering Runners](https://docs.gitlab.com/runner/register/index.html#docker)

1. 注册Runner  
   `docker run --rm -t -i --link gitlab:gitlab -v  ~/data/gitlab/runner/config:/etc/gitlab-runner --name gitlab-runner-register gitlab/gitlab-runner register`
2. 输入gitlab url  
   `Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com ):`  
   `https://gitlab.com`
3. 输入gitlab-ci token  
   `Please enter the gitlab-ci token for this runner:`  
   `4mjBGibJvizkeVuk9sv1`
4. 输入runner描述  
   `Please enter the gitlab-ci description for this runner::`  
   `golang group runner`
5. 输入runner tag  
   `Please enter the gitlab-ci tags for this runner (comma separated):`  
   `dev`
6. 选择runner executor  
   `Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:`  
   `docker`  
   [Gitlab Runner Executors](https://docs.gitlab.com/runner/executors/README.html)
7. 默认docker executor使用镜像  
   `Please enter the Docker image (eg. ruby:2.1):`  
   `golang:latest`

### Runner使用
Runner安装和注册完成之后在Gitlab上可以看到当前注册的Runner
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-ci-runner.png?raw=true)

### Runner配置
注册完Runner之后可以修改Runner配置来提高CI Pipeline的执行速度

```
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "golang project runner"
  url = "http://gitlab.local.com/"
  token = "e863c894d4950bfeed8c2cb734b6fd"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "golang:latest"    #executor默认使用的镜像
    privileged = false  
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    pull_policy = "if-not-present"   #很重要 可以保证docker executor使用的镜像优先从本地仓库拉取
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

# 三. Pipelines
添加完`.gitlab-ci.yml`文件以及配置完Runner后，每次提交新的代码到Gitlab会自动触发CI Pipeline，所有的Pipeline在Pipelines下可以查看到
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/gitlab-ci-pipelines.png?raw=true)

# 四. 常见问题
在研究Gitlab CI和CD配置的时候遇到了很多坑，因为使用的是Docker部署在同一个物理机上，中间遇到的比较大的问题是网络相关的问题。

1. Gitlab配置本地域名解析的时候不要使用`127.0.0.1`而是要使用真实的本机iP例如`192.168.0.91`，否则docker容器实例会报无法访问到127.0.0.1的Gitlab服务。
2. Gitlab Runner容器启动要使用`--link gitlab:gitlab`否则会导致Gitlab Runner容器无法访问Gitlab服务。
3. `.gitlab-ci.yml`配置使用bash不要使用sh，否则会报`/bin/sh: 1: syntax error: "(" unexpected `错误，因为是linux将sh指向了dash而不是bash。


