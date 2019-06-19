---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Python
---

# 一. datetime
Python中处理时间最重要的一个Module就是datetime

引用：import datetime
常用的类：
1. datetime.date: 代表日期（year, month, day）
2. datetime.time: 代表一天内时间（hour, minute, second, microsecond）
3. datetime.datetime: 代表完整的时间，也就是上面说的date + time（year, month, day, hour, minute, second, microsecond）
4. datetime.timedelta：代表一段时间差，支持时间维度的运算（支持和date, datetime加减运算）
5. datetime.tzinfo：处理和时区相关，是一个abstract class，使用时必须继承实现
6. datetime常见用法：
    * 获取今天0点的日期：datetime.date.today()
    * 获取昨天0点的日期：datetime.date.today() - datetime.timedelta(days=1)
    * 获取当前时间：datetime.datetime.today()
    * 获取昨天当前时间：datetime.datetime.today - datetime.timedelta(days=1)

# 二. time
提供类似C语言中time.h提供的接口，提供了一个与时间相关的工具

引用：import time
常用形式：
1. UNIX timestamp: 时间戳（相对于1970.1.1:00:00:00的秒数）
    struct_time:
    * tm_year: 年份，例如2014
    * tm_mon: 月份，range [1, 12]
    * tm_mday: 天，range [1, 31]
    * tm_hour: 小时，range [0, 23]
    * tm_min: 分钟，range [0, 59]
    * tm_sec: 秒，range [0, 61] -> 为啥是61秒？官方说手册的解释：The range really is 0 to 61; this accounts for leap seconds and the (very rare) double leap seconds. leap second是闰秒的意思，绝大多数情况都是59秒，详见[http://zh.wikipedia.org/wiki/%E9%97%B0%E7%A7%92](http://zh.wikipedia.org/wiki/%E9%97%B0%E7%A7%92)
    * tm_wday: 星期几，range [0, 6] 星期一是0
    * tm_yday, 一年当中第几天，range [1, 366]
    * tm_isdst, 夏令时, 0, 1, -1

# 三. 相互转换
1. datetime -> time.struct_time
2. datetime.datetime.timetuple()
    datetime.date.timetuple()
3. time.struct_time -> timestamp
    time.mktime(struct_time)
4. timestamp -> datetime.datetime
5. datetime.datetime.fromtimestamp(timestamp)
    datetime.datetime.utcfromtimestamp(timestamp)
    datetime.datetime -> string
    datetime.datetime.strptime()
6. string -> datetime.datetime
    datetime_obj.strftime()
    time.struct_time -> string
    time.strptime() string -> time.struct_time
    time.strftime()

# 四. 常见例子
1. datetime.datetime.today().strftime("%Y-%m-%d %H:%M:%S")  ->   "2016-06-20 15:34:45"
2. 获取今天0点0分时间戳：datetime.date.today().strftime("%s”)

