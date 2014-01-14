---
layout: post
title: "Hadoop rpc优化 参数优化"
description: ""
category: hadoop 
tags: [hadoop, rpc]
---
{% include JB/setup %}



`Hadoop rpc优化- 高并发请求`上线后新问题
===

hdfs启动后一切正常，当运维将参数`?`改回后，突然出现了102个deadnode,且这些datanode的日志中都存在如下类似的记录：

```java
java.io.IOException: Call to /10.148.xxx.xxx:9005 failed on local exception: java.io.EOFException
        at org.apache.hadoop.ipc.Client.wrapException(Client.java:783)
        at org.apache.hadoop.ipc.Client.call(Client.java:746)
        at org.apache.hadoop.ipc.RPC$Invoker.invoke(RPC.java:230)
        at $Proxy4.blocksBeingWrittenReport(Unknown Source)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.register(BPServiceActor.java:563)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.connectToNNAndHandshake(BPServiceActor.java:229)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.run(BPServiceActor.java:597)
        at java.lang.Thread.run(Thread.java:662)
Caused by: java.io.EOFException
        at java.io.DataInputStream.readInt(DataInputStream.java:375)
        at org.apache.hadoop.ipc.Client$Connection.receiveResponse(Client.java:537)
        at org.apache.hadoop.ipc.Client$Connection.run(Client.java:475)
```


临时方案
---
重启有问题的datanode，其实也可以通过-refreshNamenodes datanodehost:port恢复

问题分析
===

经过小伙伴们艰苦的排查，问题的真相终于出来了，下面说下分析过程

现象：datanode dead nodes数量从9 变成了104.
通过查看这100多个datanode日志，发现这些datanode中都在register方法中出现了`ＥＯＦＥxception`， datanode 并没有处理这种异常，这会导致BpOfferService直接退出。

datanode在read时出现EOFException 说明namenode主动关闭了连接。 为什么一定是namenode主动关闭了连接呢， 因为datanode如果自己关闭了连接，那异常必然不是EOFException.

根据这次server的优化，很快将目光集中在listener线程 清理超时连接上， 从server的代码上也确实只有这一个地方是服务端主动触发连接关闭。

接下来我们猜测 由于callqueue在datanode启动期间满了，所以reader线程会Ｗaiting在callqueue上， 而register调用（具体是register 和blocksBeingWrittenReport 两个)当被放入队列之后 才会将conn上的rpccount变量+1, 


```java
callQueue.put(call);              // queue the call; maybe blocked here
incRpcCount();
```

那就会有个很大的问题： 如果这个请求 等待了20s 才进入队列， 那在等待过程中 ，这个连接极有可能已经被listener线程当成空闲连接（idletime > 20s, rpccount ==0) 关闭掉了。 这里需要注意的是虽然从连接关闭了，但call还是可以放到callqueue中， 然后被handler线程处理。

然后我们分析handler线程的日志，因为如果连接已经被关闭了， 那么handler线程在往一个closed的channel中写数据时， 必然会失败，那就一定会产生一条output error的日志。 可当我们统计output error日志时，发现根本没有register方法的output error日志，blocksBeingWrittenReport的output error也只有4条，这与100多个deadnode数量不符。 

这让小伙伴们有些抓狂， 服务端除了这个日志没有，其他一切和我们的猜想是一致的， 难道是logging出了问题？ 难道是 连接中的responsequeue 大于1 , 处理交由responder线程处理？ 可不管通过代码分析还是 在测试集群重现， 这两个问题都可以排除。


本打算把rpc count +1的代码放在blockingqueue 前修改一把提个patch ，但就像王林大师说的， 问题没有完全重现出来，这个改动未必可以解决问题。
并且这无法说服自己啊。。

又把server的代码梳理了几遍， 终于发现了之前一直疏忽的一个问题： 很有可能reader线程 没有放到queue之前就返回了， 就是说datanode的rpc请求的tcp包到达了namenode， 但根本就没有放到callqueue中。 

思路打开了， 问题也迅速定位到了 reader线程的两个可疑点：

当callqueue满了情况下，reader线程的处理速度自然会下降，虽然有很多的datanode发了请求过来， 并且channel已经被reader线程select出来， 但当处理到这个channel时，这个channel可能早已经被listener线程根据超时关闭掉了。 那当判断key.isValid()时 就会返回false， 根本就进不去doRead方法， 

```java
            try {
              readSelector.select();
              processNewChannels();             

              Iterator<SelectionKey> iter = readSelector.selectedKeys().iterator();
              while (iter.hasNext()) {
                key = iter.next();
                iter.remove();
                if (key.isValid()) {
                  if (key.isReadable()) {
                    doRead(key);
                  }
                }
                key = null;
              }
            }

```

另外一个可能是，即使这个channel已经进入了doRead方法，如果此时被listener线程超时关闭了，readAndProcess线程就会由于ChannelClosedException退出， 


```java
void doRead(SelectionKey key) throws InterruptedException {
      int count = 0;
      Connection c = (Connection)key.attachment();
      if (c == null) {
        return;  
      }
      c.setLastContact(System.currentTimeMillis());
      
      try {
        count = c.readAndProcess();
      } catch (InterruptedException ieo) {
        LOG.info(getName() + ": readAndProcess caught InterruptedException", ieo);
        throw ieo;
      } catch (Exception e) {
        LOG.info(getName() + ": readAndProcess threw exception " + e + ". Count of bytes read: " + count, e);
        count = -1; //so that the (count < 0) block is executed
      }
      if (count < 0) {
        if (LOG.isDebugEnabled())
          LOG.debug(getName() + ": disconnecting client " + 
                    c.getHostAddress() + ". Number of active connections: "+
                    getNumOpenConnections());
        closeConnection(c);
        c = null;
      }
      else {
        c.setLastContact(System.currentTimeMillis());
      }
    }   

```

这两种情况都不会将这个请求放到callqueue中。 所有的现象都可以解释得通了。

对于第一种情况由于没有任何日志无法从日志上得到验证

第二种情况，可以在rpc log中统计"readAndProcess threw exception"得到部分验证，但由于缺少详细信息，只能说明 reader线程在read过程中确实存在channel被关闭的情况。 

最终还是通过在dw测试服务器上 验证了上面的想法。

解决方案
===

通过前面的分析可以看出，即使将rpccount+1 放到blockingqueue.put前， 也无法解决问题。 facebook的做法（在read 到header时就做rpc count +1) 也不能彻底解决问题。 社区的方案很巧合的和我们的做法是一样的， 如果配置不当 也可能会有同样的问题。

其实解决方案也比较简单，就是将timeout（`ipc.client.connection.maxidletime`）时间设置得更大一些（建议15分钟，可以更长） 这样就基本上可以解决误杀连接的问题了。

考虑到程序中会乘2,


```java
this.maxIdleTime = 2*conf.getInt("ipc.client.connection.maxidletime", 1000);
```
所以将以将`ipc.client.connection.maxidletime`设置为450000

从问题的根源来看，是由于优化后Listener能主动关闭以前无法关闭的死链接了，但怎么判断哪些是死链接的判断依据不太明确导致的，优化前这些链接是永远无法关闭，现在改成多久以后关闭比较合适的问题

按照rpc Client中的超时逻辑，client是认为15分钟（45次＊20s，并且这里的45次和20s都是在rpc Client代码中写死的）后如果仍然无法连接server则关闭client的connection，所以在这段时间里，server如果主动关闭（比如这次的这种情况）都会出现这次类似的情况，所以调成15分钟我觉得是一个比较合理的方式。优化前是永远无法关闭，优化后即使调大了这个配置，15分钟后也能关闭，所以这个调整是合理的。（相当于说以前idle time这个配置是不起作用的，看上去10s到15min跨度很大，但10s不起作用，相当于就是LONG MAX）


并且这个时间即使很长也没有关系， 最差的情况也比优化前要好，因为优化前由于程序的bug， 超时的连接根本就无法关闭， 这也导致了之前namenode中出现了2w左右的一直无法关闭的连接。

ipc.client.connection.maxidletime参数值的讨论
---

nn上的每个改动哪怕是配置的修改都需要相当谨慎，`ipc.client.connection.maxidletime`从10000改到450000 的建议还是有同学认为步子迈的太大，不是很放心。为了消除大家的疑虑，只能在测试上做更多的功课：

测试还是与上次连接压测的方式相同，参数为：

```bash
    /home/hadoop/hadoop-current/bin/hadoop jar /home/hadoop/hadoop-current//sgsimulator_connection.jar -Dsangesimu.threads.per.task=640 -Dsangesimu.op.interval=0 -Dmapred.map.tasks=700 -Dsangesimu.runtime.sec=1200 /tmp/sg
``` 
两次测试的唯一的不同就是namenode中ipc.client.connection.maxidletime 参 数不 同 （10000 vs 450000)。

|         | 10000           | 450000  |
| ------------- |:-------------:| -----:|
| 平均响应时间      | 2300 | 2236 |
| timeout error count      | 22914      |   18807 |
| namenode最 大的establish数 | 395058      |    432840 |
| namenode最 大的SYN_RECV数|1 |1|



<table>
<tr>
    <td><img src=/assets/images/client.connection.maxideltime/image001.jpg /></td>
    <td><img src=/assets/images/client.connection.maxideltime/image002.jpg /></td>
</tr>
<tr>
    <td><img src=/assets/images/client.connection.maxideltime/image003.jpg /></td>
    <td><img src=/assets/images/client.connection.maxideltime/image004.jpg /></td>
</tr>
<tr>
    <td><img src=/assets/images/client.connection.maxideltime/image005.jpg /></td>
    <td><img src=/assets/images/client.connection.maxideltime/image006.jpg /></td>
</tr>
<tr>
    <td><img src=/assets/images/client.connection.maxideltime/image007.jpg /></td>
    <td><img src=/assets/images/client.connection.maxideltime/image008.jpg /></td>
</tr>
</table>

注：PEAK_THREAD_COUNT统计方式：

```java
/**
   * An updater that tracks the peak thread count. If the number exceeds
   * the threshold, throw a RuntimeException to failed this task.
   */
  class ThreadCountUpdater {
    private ThreadMXBean threadMXBean; 
    private int preThreadCount = 0;
    private Counters.Counter threadCountCounter;
    
    public ThreadCountUpdater() {
    }
    
    void updateCounters() {
      if (threadCountCounter == null || threadMXBean == null) {
        threadMXBean = ManagementFactory.getThreadMXBean();
        threadCountCounter = counters.findCounter(Counter.PEAK_THREAD_COUNT);
      }
      int newThreadCount = threadMXBean.getThreadCount();
      if (preThreadCount < newThreadCount) {
        threadCountCounter.increment(newThreadCount - preThreadCount);
        preThreadCount = newThreadCount;
      }
      
      int threadCountLimit = conf.getInt("mapred.task.threadcount.limit", -1);
      if (threadCountLimit != -1 && preThreadCount > threadCountLimit) {
        throw new RuntimeException("peak thread count is exceeded: peak= " + preThreadCount
            + " limit= " + threadCountLimit);
      }
    }
  }
```
可以看到PEAK_THREAD_COUNT不能精确的表示集群某个时刻的并发度，但可以从宏观上体现集群的并发度。

测试总结
---

通过job的各项指标的细节比较，发现参数调整前后job的 行为基本一致。对job的 调度、连接及rpc响应时间 基本没有影响，

