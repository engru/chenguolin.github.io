---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - InfluxDB
---

#### 1.CQ创建
```
CREATE CONTINUOUS QUERY xxxx_cq ON media_quality 
RESAMPLE FOR 20m 
BEGIN SELECT 
sum(play_num) AS play_num, 
... 
INTO xxx.rp_10m.yyy 
FROM xxx.autogen.yyy GROUP BY time(10m), * 
END
```

#### 2.创建数据库  
$ create database live_monitor;
    
#### 3.删除数据库  
$ drop database media_quality;

#### 4.删除所有的measurements  
$ DROP SERIES FROM /.*/

#### 5.查看某个measurement的所有tags  
$ show tag keys from meipai_pullstream_quit_live_play_statistic;

#### 6.查看某个measurement的所有fields  
$ show field keys from meipai_pullstream_quit_live_play_statistic;

#### 7.查看CQ  
$ SHOW CONTINUOUS QUERIES

#### 8.查看RP  
$ SHOW RETENTION POLICIES

#### 9.启动influxdb  
$ sudo /usr/bin/influxd -config /etc/influxdb/influxdb.conf 1>/etc/influxdb/logs/stdout 2>/etc/influxdb/logs/stderr &

