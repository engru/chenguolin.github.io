---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - InfluxDB
---

# 一. 数据库配置
1. 查询: show database
2. 创建: CREATE DATABASE {database_name} [WITH [DURATION <duration>] [REPLICATION <n>] [SHARD DURATION <duration>] [NAME <retention-policy-name>]
3. 删除: DROP DATABASE {database_name}

# 二. RETENTION POLICY
1. 查询: SHOW RETENTION POLICIES
2. 创建: CREATE RETENTION POLICY {retention_policy_name} ON {database_name} DURATION {duration} REPLICATION {n} [SHARD DURATION {duration}] [DEFAULT]
    * CREATE RETENTION POLICY rp_10m on hubble duration 9408h0m0s REPLICATION 1 SHARD DURATION 168h0m0s
    *  CREATE RETENTION POLICY rp_1h on hubble duration 8760h0m0s REPLICATION 1 SHARD DURATION 168h0m0s
    *  CREATE RETENTION POLICY rp_6h on hubble duration 52560h0m0s REPLICATION 1 SHARD DURATION 168h0m0s
3. 修改: ALTER RETENTION POLICY {rp_name} ON {database_name} DURATION {duration} REPLICATION {n} SHARD DURATION {duration} DEFAULT
4. 删除: DROP RETENTION POLICY {rp_name} ON {database_name}

# 三. CONTINUOUS QUERY
1. 查询: SHOW CONTINUOUS QUERY
2. 删除: DROP CONTINUOUS QUERY {cq_name} ON {database_name}

# 四. API
InfluxDB API提供了较简单的方式用于数据库交互。该API使用了HTTP的方式，并以JSON格式进行返回。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/influxdb-endpoint.png?raw=true)

## ① /ping
`/ping` 支持GET和HEAD，都可用于获取指定信息。

定义：
1. GET [http://localhost:8086/ping](http://localhost:8086/ping)
2. HEAD [http://localhost:8086/ping](http://localhost:8086/ping)

获取InfluxDB版本信息  
$ curl -sl -I [http://localhost:8086/ping](http://localhost:8086/ping)
```
HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: ebe357b8-ea19-11e7-8001-000000000000
X-Influxdb-Version: v1.3.6
Date: Tue, 26 Dec 2017 08:51:11 GM
```

## ② /query
`/query` 支持GET和POST的HTTP请求。可用于查询数据和管理数据库、rp、users。

定义：
1. GET [http://localhost:8086/](http://localhost:8086/ping)query
2. HEAD [http://localhost:8086/](http://localhost:8086/ping)query

1. 使用SELECT查询数据  
   $ curl -G '[http://localhost:8086/query?db=mydb](http://localhost:8086/query?db=mydb)' --data-urlencode 'q=SELECT * FROM "mymeas”'  
   {"results":[{"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2016-05-20T21:30:00Z",12,"1",null],["2016-05-20T21:30:20Z",11,"2",null],["2016-05-20T21:30:40Z",18,null,"1"],["2016-05-20T21:31:00Z",19,null,"3"]]}]}]}

2. 使用INTO字段  
   $ curl -XPOST '[http://localhost:8086/query?db=mydb](http://localhost:8086/query?db=mydb)' --data-urlencode 'q=SELECT * INTO "newmeas" FROM "mymeas”'  
   {"results":[{"series":[{"name":"result","columns":["time","written"],"values":[["1970-01-01T00:00:00Z",4]]}]}]}

3. 创建数据库  
   $ curl -XPOST '[http://localhost:8086/query](http://localhost:8086/query)' --data-urlencode 'q=CREATE DATABASE "mydb”'  
   {"results":[{}]}

Query参数说明：
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/influxdb-query-args.png?raw=true)

1. 使用http认证来创建数据库  
$ curl -XPOST '[http://localhost:8086/query?u=myusername&p=mypassword](http://localhost:8086/query?u=myusername&p=mypassword)' --data-urlencode 'q=CREATE DATABASE "mydb"'  
   {"results":[{}]}

2. 使用基础认证来创建数据库  
$ curl -XPOST -u myusername:mypassword '[http://localhost:8086/query](http://localhost:8086/query)' --data-urlencode 'q=CREATE DATABASE "mydb”'  
   {"results":[{}]}

## ③ /write
`/wirte` 只支持POST的HTTP请求，使用该Endpoint可以写数据到已存在的数据库中。

定义：POST [http://localhost:8086/write](http://localhost:8086/write)

write参数说明  
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/influxdb-write-args.png?raw=true) 

数据请求体：--data-binary '< Data in Line Protocol format >'

1. 使用秒级的时间戳，将一个point写入数据库mydb  
$ curl -i -XPOST "[http://localhost:8086/write?db=mydb&precision=s](http://localhost:8086/write?db=mydb&precision=s)" --data-binary 'mymeas,mytag=1myfield=90 1463683075'

2. 将一个point写入数据库mydb，并指定RP为myrp  
$ curl -i -XPOST "[http://localhost:8086/write?db=mydb&rp=myrp](http://localhost:8086/write?db=mydb&rp=myrp)" --data-binary 'mymeas,mytag=1 myfield=90'

3. 使用HTTP认证的方式，将一个point写入数据库mydb  
$ curl -i -XPOST "[http://localhost:8086/write?db=mydb&u=myusername&p=mypassword](http://localhost:8086/write?db=mydb&u=myusername&p=mypassword)" --data-binary 'mymeas,mytag=1 myfield=91'

4. 使用基础认证的方式，将一个point写入数据库mydb  
$ curl -i -XPOST -u myusername:mypassword "[http://localhost:8086/write?db=mydb](http://localhost:8086/write?db=mydb)" --data-binary 'mymeas,mytag=1 myfield=91'

5. 写多个points到数据库中,需要使用新的一行  
$ curl -i -XPOST "[http://localhost:8086/write?db=mydb](http://localhost:8086/write?db=mydb)" --data-binary 'mymeas,mytag=3 myfield=89  
  mymeas,mytag=2 myfield=34 1463689152000000000'

6. 通过导入文件的形式，写入多个points。需要使用@来指定文件  
$ curl -i -XPOST "[http://localhost:8086/write?db=mydb](http://localhost:8086/write?db=mydb)" --data-binary @data.txt

文件内容如下
```
mymeas,mytag1=1 value=21 1463689680000000000
mymeas,mytag1=1 value=34 1463689690000000000
mymeas,mytag2=8 value=78 1463689700000000000    
mymeas,mytag3=9 value=89 1463689710000000000
```

响应的状态码  
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/influxdb-http-code.png?raw=true)

