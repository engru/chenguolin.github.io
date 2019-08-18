---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Hive
---

# 一. 安装
1. hive安装：brew install hive
2. mysql安装：brew install mysql
3. 启动mysql：bash mysql.server start

# 二. 元数据库配置
Hive默认用derby作为元数据库。这里我们用mysql来存储元数据，下面作一些初始化配置
1. 登录mysql：mysql -u root
2. 创建数据库：create database metastore;
3. 创建新的用户：create user 'hive'@'localhost' identified by '123456’;
4. 修改用户权限：grant select,insert,update,delete,alter,create,index,references on metastore.* to 'hive'@'localhost’;
5. 刷新权限：flush privileges;

# 三. 配置Hive
1. 进入Hive的安装目录,创建hive-site.xml文件
   + cd /usr/local/Cellar/hive/2.1.1/libexec/conf
   + cp hive-default.xml.template hive-site.xml
2. 修改hive-site.xml文件，找到对应的property并修改其值
3. Hadoop core-site添加如下配置给当前用户授权访问hdfs
4. 配置hive日志目录  
   + cp hive-log4j.properties.template hive-log4j.properties
   + vi hive-log4j.properties
   + hive.log.dir=/usr/local/hive/log     //配置hive log的目录
5. 拷贝mysql-connector到hive，给Hive的lib目录下拷贝一个mysql-connector
   + curl -L '[http://www.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.42.tar.gz/from/http://mysql.he.net/'](http://www.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.42.tar.gz/from/http://mysql.he.net/'); | tar xz
   + cp mysql-connector-java-5.1.42/mysql-connector-java-5.1.42-bin.jar /usr/local/Cellar/hive/2.1.1/libexec/lib/

# 四. 初始化元数据库
1. 初始化metastore库：schematool -initSchema -dbType mysql
2. 登录mysql：mysql -u hive -p123456
3. 使用metastore数据库：use metastore
4. 查看表：show tables

# 五. HDFS创建目录
1. HDFS上建立/tmp和/usr/hive/warehouse目录，并赋予组用户写权限，配置Hive默认的数据文件存放目录
   + hadoop dfs -mkdir -p hdfs://localhost:9000/user/hive/warehouse
   + hadoop dfs -mkdir -p hdfs://localhost:9000/user/hive/tmp
   + hadoop dfs -chmod 777 hdfs://localhost:9000/user/hive/warehouse
   + hadoop dfs -chmod 777 hdfs://localhost:9000/user/hive/tmp

# 六. 启动服务
1. 启动metastore服务: hive —service metastore &
2. 启动hiverserver2服务: hive --service hiveserver2
3. 可以开始使用jdbc方式连接hivesever进行读写操作

# 七. 客户端连接代码
```
package com.test.hive.client;

import java.sql.SQLException;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.Statement;
import java.sql.DriverManager;

public class HiveJdbcClient {
    private static String driverName = "org.apache.hive.jdbc.HiveDriver";
    
    /**
     * @param args
     * @throws SQLException
     */
    public static void main(String[] args) throws SQLException {
    try {
        Class.forName(driverName);
    } catch (ClassNotFoundException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
        System.exit(1);
    }
    
    // 1. connect to hive2
    Connection con = DriverManager.getConnection("jdbc:hive2://localhost:10000/default", "", "");
    Statement stmt = con.createStatement();
    
    // 2. create table
    String tableName = "testHiveDriverTable";
    stmt.execute("drop table if exists " + tableName);
    stmt.execute("create table " + tableName + " (key int, value string)");

    // 3. show tables
    String sql = "show tables '" + tableName + "'";
    System.out.println("Running: " + sql);
    ResultSet res = stmt.executeQuery(sql);
    if (res.next()) {
        System.out.println(res.getString(1));
    }

    // 4. describe table
    sql = "describe " + tableName;
    System.out.println("Running: " + sql);
    res = stmt.executeQuery(sql);
    while (res.next()) {
        System.out.println(res.getString(1) + "\t" + res.getString(2));
    }
    
    // 5. select * query
    sql = "select * from " + tableName;
    System.out.println("Running: " + sql);
    res = stmt.executeQuery(sql);
    while (res.next()) {
        System.out.println(String.valueOf(res.getInt(1)) + "\t" + res.getString(2));
    }
    
    // 6. regular hive query
    sql = "select count(1) from " + tableName;
    System.out.println("Running: " + sql);
    res = stmt.executeQuery(sql);
    while (res.next()) {
        System.out.println(res.getString(1));
    }
}
```

