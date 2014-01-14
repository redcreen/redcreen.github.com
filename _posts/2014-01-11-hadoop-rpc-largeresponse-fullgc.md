---
layout: post
title: "Hadoop rpc优化 FullGC"
description: "fullgc cmsgc large response "
category: hadoop
tags: [hadoop, rpc, java, gc]
---
{% include JB/setup %}

最近namenode频繁出现fullgc，对于一个Heap Size达到150G的jvm， fullgc一次就需要15~20分钟，在fullgc期间，namenode会暂停对外服务， 整个集群不可用。

根据fullgc的日志，主要分为三种：

+ promotion failed

```
2013-10-10T06:11:47.654+0800: 1444544.329: [GC 1444544.329: [ParNew (promotion failed): 3774912K->3682402K(3774912K), 1.2662220 secs]1444545.596: [CMS: 128047712K->89062768K(153092096K), 811.7515150 secs] 131716601K->89062768K(156867008K), [CMS Perm : 22265K->22258K(37136K)], 813.0179170 secs] [Times: user=815.14 sys=2.21, real=812.89 secs]
```

+ concurrent mode failure

```
2013-10-09T17:03:50.334+0800: 1397267.009: [GC 1397267.010: [ParNew (promotion failed): 3647305K->3684179K(3774912K), 0.8413060 secs]1397267.851: [CMS2013-10-09T17:04: 
21.516+0800: 1397298.192: [CMS-concurrent-sweep: 30.519/31.363 secs] [Times: user=34.84 sys=0.18, real=31.35 secs] 
(concurrent mode failure): 135995125K->89169200K(153092096K), 840.6461240 secs] 139547450K->89169200K(156867008K), [CMS Perm : 22262K->22256K(37136K)], 841.4875980 se 
cs] [Times: user=841.80 sys=1.63, real=841.35 secs] 

```

+ Full GC

```
2013-10-24T09:52:00.676+0800: 1113752.804: [GC 1113752.804: [ParNew: 3600675K->223413K(3774912K), 0.1213160 secs] 116541912K->113173861K(156867008K), 0.1214370 secs] [Times: user=2.26 sys=0.01, real=0.12 secs]
2013-10-24T09:52:04.494+0800: 1113756.622: [GC 1113756.622: [ParNew: 3578933K->274534K(3774912K), 0.1170330 secs] 116529381K->113234917K(156867008K), 0.1171570 secs] [Times: user=2.21 sys=0.00, real=0.12 secs]
2013-10-24T09:52:06.488+0800: 1113758.616: [Full GC 1113758.616: [CMS: 112960382K->94922308K(153092096K), 862.4321220 secs] 115014899K->94922308K(156867008K), [CMS Perm : 22261K->22253K(37240K)], 862.4322660 secs] [Times: user=859.43 sys=2.29, real=862.30 secs]
```

promotion failed 及 concurrent mode failure 产生的原因见这篇文章：[java-garbage-collection-fail]

通过调整JVM参数，可以在一定程度上解决FullGC的问题： 
通过将-XX:CMSInitiatingOccupancyFraction 从85 设置到 80， 小伙伴@葛聂已经验证了 fullgc频率会降低40%左右。

但问题的根源还是部分客户端调用的rpc接口会返回大对象（100M+), 而hadoop rpc机制导致了当有大的response对象时，内存分配会存在很严重的问题， 给GC及Mem Bandwidth都带来了很大的压力。 先看下关键代码：

```java
public void run() {
      ...
      ByteArrayOutputStream buf = new ByteArrayOutputStream(initRespBufSize);
      while (running) {
        try {
          Call call = callQueue.take(); // pop the queue; maybe blocked here
          ...
          value = call(call.param, call.timestamp);             // make the call   
          buf.reset();
          DataOutputStream out = new DataOutputStream(buf);
          ... 
          value.write(out);//write value to outputstrem
          call.setResponse(ByteBuffer.wrap(buf.toByteArray()));
          responder.doRespond(call);
        } 
        ...
      }
    }
```

假设value序列化后的长度是1024*1024*129 (byte）， 即129MB， initRespBufSize初始值为1M， 那内存分配将是： 1M， 2M， 4M ，8M， 16M ，32M， 64M ，128M , 256M , 256M，共767M, 是129M的将近6倍， 而这767M的内存都是在序列化过程中快速分配的，这对GC的压力是非常大的。 （内存超过多少M就晋升了呢？）

rpc的实现原理决定了对于结果对象，先要在内存序列化完成后才会通过channal发送，对于大对象的结果， 在内存中缓存序列化后的结果，内存开销自然不小。 最近发现hbase的rpc也有同样的问题。

优化
---

虽然rpc不太适用这种场景，也应尽可能将这种调用改成streaming的方式。 但rpc本身还是有很大的优化空间。 

主要的优化思路是：

+ 序列化避免出现大量的中间结果（减少不必要的内存开销）
+ 避免一次分配一个大的内存，而是分配若干小的内存。（避免内存碎片导致的大内存申请失败） 

优化代码：

+ 实现一个CompositeByteBuffer，用于封装内存分配）

```java
package org.apache.hadoop.ipc;

import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

class CompositeByteBuffer {
  private final List<ByteBuffer> components = Collections
      .synchronizedList(new ArrayList<ByteBuffer>(1));

  private int componentMaxSize = 1024 * 1024 * 4;
  private int componentMinSize = 1024 * 4;
  private int offset = 0;
  private int capacity = 0;
  private int index= 0;
  
  public CompositeByteBuffer(int initsize, int componentSize) {
    this.componentMinSize = initsize;
    this.componentMaxSize = componentSize;
    components.add(alloc());
  }

  public void writeByte(int b) {
    ensureWrite(1);
    getNextAvailableComponent().put((byte)b);
    offset++;
  }

  public void writeBytes(byte[] b, int off, int len) {
    ensureWrite(len);
    while (true) {
      ByteBuffer bb = getNextAvailableComponent();
      int remain = bb.remaining();
      if (len <= remain) {
        bb.put(b, off, len);
        offset = offset + len;
        return ;
      } else {
        bb.put(b, off, remain);
        off = off + remain;
        len = len - remain;
        offset = offset + remain;
      }
    }
  }
  
  public int readableBytes() {
    return offset;
  }
  
  public ByteBuffer nioBuffer(int i) {
    return components.get(i);
  }
  
  public ByteBuffer[] nioBuffers (){
    return components.toArray(new ByteBuffer[components.size()]);
  }
  
  public int nioBufferCount () {
    return components.size();
  }

  private void ensureWrite(int len) {
    if (offset + len <= capacity) {
      return ;
    } else {
      int remaining = 0;
      while (true) {
        ByteBuffer currentbb = components.get(components.size() -1); 
        remaining = remaining + currentbb.remaining();
        if (remaining >= len) {
          return ;
        } else {
          components.add(alloc());
          continue;
        }
      }
    }
  }

  private ByteBuffer getNextAvailableComponent() {
    while (true) {
      ByteBuffer bb = components.get(index);
      if (bb.remaining() > 0) {
        return bb;
      } else {
        index++;
        continue;
      }
    }
  }

  private ByteBuffer alloc() {
    int size;
    if (capacity > 1024 * 1024) {
      size = componentMaxSize;
    } else {
      size = Math.min(componentMaxSize, componentMinSize << components.size());
    }
    this.capacity = this.capacity + size;
    return ByteBuffer.allocate(size);
  }
  public static void  main(String[] args) {
    CompositeByteBuffer cb = new CompositeByteBuffer(1024*16, 1024 * 1024 * 4);
    byte[] bytes = new byte[1024*1024];
    for (int i=0;i<8;i++){
      cb.writeBytes(bytes, 0, bytes.length);
    }
    System.out.println(cb.readableBytes()); 
    System.out.println(cb.nioBufferCount()); 
    for (ByteBuffer bb : cb.nioBuffers()){
      System.out.println(bb.capacity() +":"+bb.remaining());
    }
  }
}
```


+ 构造DataoutputStream

```java
final CompositeByteBuffer cbuf = new CompositeByteBuffer(1024 * 16, 1024 * 1024 * 4);
          
          OutputStream buf = new OutputStream() {
            @Override
            public void write(int b) throws IOException {
              cbuf.writeByte(b);
            }
            @Override
            public void write(byte[] b, int off, int len) throws IOException {
              cbuf.writeBytes(b, off, len);
            }
          };
          
          DataOutputStream out = new DataOutputStream(buf);
```

+ 大的response 拆成多个小的response.

```java
//
    // Enqueue a response from the application.
    //
    void doRespond(Call call) throws IOException {
      synchronized (call.connection.responseQueue) {
        for (int i =0 ; i< call.response.nioBufferCount(); i++) {
          ByteBuffer buffer = call.response.nioBuffer(i);
          Call ci = new Call(call.id, call.param, call.connection, call.responder);
          buffer.flip();
          ci.setTrunk(buffer);
          if (i < call.response.nioBufferCount() -1) {
            ci.lastTrunk = false;
          }
          ci.timestamp = call.timestamp;
          call.connection.responseQueue.addLast(ci);
        }
        boolean flag = false;
        if (call.response.nioBufferCount() > 1) {
          flag = true;
        }
        if (call.connection.responseQueue.size() == call.response.nioBufferCount()) {
          processResponse(call.connection.responseQueue, true);
        }
        call.response = null;
        if (flag) { //multy chucks should register channel.
          registerWriteSelector(call.connection.channel, call, null);
        }
      }
    }
```

代码详见：[Server.java][server.code]

[patch下载][patch]




[understanding_cms_gc_logs]: https://blogs.oracle.com/poonam/entry/understanding_cms_gc_logs
[when_the_sum_of_the]: https://blogs.oracle.com/jonthecollector/entry/when_the_sum_of_the
[what_the_heck_s_a]: https://blogs.oracle.com/jonthecollector/entry/what_the_heck_s_a
[java-garbage-collection-fail]: /java/2014/01/14/java-garbage-collection-fail
[server.code]: https://github.com/redcreen/hadoop-19-common/blob/master/src/core/org/apache/hadoop/ipc/Server.java
[patch]: /assets/code/rpc1.patch