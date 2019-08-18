---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Storm
---

1. 滑动窗口在监控和统计应用的场景比较广泛，比如每隔一段时间(10s)统计最近30s的请求量或者异常次数，根据请求或者异常次数采取相应措施；这里说一下滑动窗口在storm中实现的原理

2. 那么如何每10s进行自动触发，storm有一个TickTuple可以满足这个要求
   ```
   "__system" component会定时往task发送 "__tick" stream的tuple 发送频率由TOPOLOGY_TICK_TUPLE_FREQ_SECS来配置
   可以在default.ymal里面配置也可以在代码里面通过getComponentConfiguration()来进行配置,
 
   public Map<String, Object> getComponentConfiguration() {
        Map<String, Object> conf = new HashMap<String, Object>(); 
        conf.put(Config.TOPOLOGY_TICK_TUPLE_FREQ_SECS, emitFrequencyInSeconds); 
        return conf; 
   }
   ```

3. 配置完成后, storm就会定期的往task发送ticktuple，只需要通过isTickTuple来判断是否为tickTuple, 就可以完成定时触发的功能 
   ```
   public static boolean isTickTuple(Tuple tuple) {
      // SYSTEM_COMPONENT_ID == "__system" 
      return tuple.getSourceComponent().equals(Constants.SYSTEM_COMPONENT_ID)
      // SYSTEM_TICK_STREAM_ID == "__tick" 
      && tuple.getSourceStreamId().equals(Constants.SYSTEM_TICK_STREAM_ID);   
   } 
   ```
