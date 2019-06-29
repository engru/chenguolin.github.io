---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Spark
---

# 一. 安装yarn伪分布式集群
## ① 创建新用户
  * 添加用户: sudo useradd -m hadoop -s /bin/bash
  * 修改密码: sudo passwd hadoop
  * 添加sudo权限: sudo adduser hadoop sudo
  * 注销选择hadoop用户登录

## ② 系统环境配置
  * sudo apt-get update
  * 安装vim: sudo apt-get install vim
  * 安装ssh: sudo apt-get install openssh-server
  * 设置ssh免登录
    * cd ~/.ssh
    * rm ./id_rsa*
    * ssh-keygen -t rsa  (一路回车即可)
    * cat ./id_rsa.pub >> ./authorized_keys

## ③ 安装java环境
  * 安装jre和jdk: sudo apt-get install openjdk-7-jre openjdk-7-jdk
  * 设置环境变量JAVA_HOME: dpkg -L openjdk-7-jdk | grep '/bin/javac': 该命令会输出一个路径，除去路径末尾的 “/bin/javac”
  * vim ~/.bashrc: 添加一行export JAVA_HOME=...
  * source ~/.bashrc
  * check java 版本: java -version

## ④ 安装hadoop2
  * 从http://mirror.bit.edu.cn/apache/hadoop/common/下载最新的稳定版本的hadoop，例如hadoop-2.7.3/hadoop-2.7.3.tar.gz
  * 安装hadoop
    * sudo tar -zxf hadoop-2.7.3.tar.gz -C /usr/local
    * cd /usr/local
    * sudo mv hadoop-2.7.3 hadoop
    * sudo chown -R hadoop hadoop

## ⑤ 配置hadoop
  * /usr/local/hadoop/etc/hadoop进入hadoop配置目录，如果没有hadoop-env.sh或yarn-env.sh需要从后缀名为hadoop-env.sh.template复制一份
    * 在hadoop-env.sh中配置JAVA_HOME
    * 在yarn-env.sh中配置JAVA_HOME
    * 修改core-site.xml
    ```
    <configuration>
        <property>
             <name>hadoop.tmp.dir</name>
             <value>file:/usr/local/hadoop/tmp</value>
             <description>Abase for other temporary directories.</description>
        </property>
        <property>
             <name>fs.defaultFS</name>
             <value>hdfs://localhost:9000</value>
        </property>
    </configuration>
    ```
    * hdfs-site.xml
    ```
    <configuration>
        <property>
             <name>dfs.replication</name>
             <value>1</value>
        </property>
        <property>
             <name>dfs.namenode.name.dir</name>
             <value>file:/usr/local/hadoop/tmp/dfs/name</value>
        </property>
        <property>
             <name>dfs.datanode.data.dir</name>
             <value>file:/usr/local/hadoop/tmp/dfs/data</value>
        </property>
     </configuration>
     ```
     * mapred-site.xml
     ```
     <configuration>
       <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
       </property>
     </configuration>
     ```
    * yarn-site.xml
    ```
    <configuration>
      <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
      </property>
    </configuration>
    ```

## ⑥ 启动hadoop
  * namenode格式化: ./bin/hdfs namenode -format
  * 修改~/.bashrc添加以下两行，并执行source ~/.bashrc
    * export HADOOP_HOME=/usr/local/hadoop 
    * export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
  * ./libexec/hadoop-config.sh 找到JAVA_HOME，在前面添加
    * export JAVA_HOME=...
  * 开启 NameNode 和 DataNode 守护进程
    * ./sbin/start-dfs.sh
  * jps命令查看是否成功启动
    * 若成功启动则会列出如下进程: “NameNode”、”DataNode” 和 “SecondaryNameNode”（如果 SecondaryNameNode 没有启动，请运行 sbin/stop-dfs.sh 关闭进程，然后再次尝试启动尝试）。如果没有 NameNode 或 DataNode ，那就是配置不成功，请仔细检查之前步骤，或通过查看启动日志排查原因。
  * 启动成功后可以通过 http://localhost:50070/查看nameNode和dataNode相关信息
  * 关闭hadoop
    * ./sbin/stop-dfs.sh
    * 第二次之后启动 hadoop，无需进行 NameNode 的初始化，只需要运行 ./sbin/start-dfs.sh 即可

## ⑦ 启动YARN
  * 启动yarn
    * ./sbin/start-yarn.sh 
    * ./sbin/mr-jobhistory-daemon.sh start historyserver  #开启历史服务器，才能在Web中查看任务运行情况
  * jps查看
    * 开启后通过 jps 查看，可以看到多了 NodeManager 和 ResourceManager 两个后台进程
  * 启动成功后可以通过页面http://localhost:8088/cluster查看集群任务的运行情况
  * 关闭yarn
    * ./sbin/stop-yarn.sh 
    * ./sbin/mr-jobhistory-daemon.sh stop historyserver

# 二. 安装Spark
## ① 下载Spark
  * wget "http://d3kbcqa49mib13.cloudfront.net/spark-2.0.0-bin-hadoop2.7.tgz"

## ② 解压到/usr/local
  * sudo tar -xvzf spark-2.0.0-bin-hadoop2.7.tgz -C /usr/local
  * cd /usr/local
  * sudo mv spark-2.0.0-bin-hadoop2.7 spark
  * sudo chown -R hadoop spark

## ③ 设置环境变化PATH
  * vim ~/.bashrc
  * export PATH=$PATH:/usr/local/hadoop/bin:/usr/local/spark/bin
  * source ~/.bashrc

## ④ 配置Spark 
  * cd /usr/local/spark/conf 
  * cp spark-env.sh.template spark-env.sh 
  * vim spark-env.sh
  ```
  配置内容如下
  export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-i386
  export HADOOP_HOME=/usr/local/hadoop   
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop=   
  SPARK_MASTER_IP=master
  SPARK_LOCAL_DIRS=/usr/local/spark
  SPARK_DRIVER_MEMORY=1G
  ```
  
## ⑤ 启动spark
  * sh /usr/local/sbin/start-all.sh
  * 启动成功后可以使用 pyspark或spark-submit
  * 同时也可以访问以下链接查看spark任务 http://master:8080

# 三. 使用Yarn运行Spark程序
  为了方便，以后我们都采用Yarn来运行Spark任务，不会再单独启动Spark

## ① 机器配置
  * sudo chown hadoop:root -R /usr/local/hadoop
  * sudo chown hadoop:root -R /usr/local/spark
  * sudo chmod 775 -R /usr/local/hadoop
  * sudo chmod 775 -R /usr/local/spark
  * bashrc配置
  ```
  export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-i386
  export HADOOP_HOME=/usr/local/hadoop   
  export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
  export HADOOP_OPTS=-Djava.library.path=$HADOOP_HOME/lib
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export PATH=$PATH:/usr/local/hadoop/bin:/usr/local/spark/bin
  export LD_LIBRARY_PATH=/usr/local/hadoop/lib/native:$LD_LIBRARY_PATH
  ```

## ② 启动yarn
  * cd /usr/local/hadoop
  * ./sbin/start-dfs.sh
  * ./sbin/start-yarn.sh
  * ./sbin/mr-jobhistory-daemon.sh start historyserver
  * jps查看进程，应该有以下几个
  ```
  16891 NodeManager
  16951 JobHistoryServer
  16502 SecondaryNameNode
  16028 NameNode
  17729 Jps
  16683 ResourceManager
  16228 DataNode
  ```

## ③ 停止yarn
  * cd /usr/local/hadoop
  * ./sbin/stop-dfs.sh
  * ./sbin/stop-yarn.sh
  * ./sbin/mr-jobhistory-daemon.sh stop historyserver
      
## ④ web界面查看
  * 查看nameNode和dataNode: http://localhost:50070/
  * 查看yarn集群任务: http://localhost:8088/cluster

# 四. 问题汇总
## ① 问题
```
Hadoop 2.x.x - warning: You have loaded library /home/hadoop/2.2.0/lib/native/libhadoop.so.1.0.0 which might have disabled stack guard.
解决方案
1. vi ~/.bashrc
2. 添加以下2行
   export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP_HOME}/lib/native
   export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib
```
