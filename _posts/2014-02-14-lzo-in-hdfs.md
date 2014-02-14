---
layout: post
title: "test lzo in hdfs"
description: ""
category: hadoop 
tags: [hadoop, native]
---
{% include JB/setup %}

一个简单的实例演示如何在ＨＤＦＳ中使用LZO压缩。

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.InputStream;
import java.io.OutputStream;
import java.lang.reflect.Field;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.compress.CompressionCodec;
import org.apache.hadoop.io.compress.CompressionCodecFactory;
import org.apache.hadoop.io.compress.CompressionInputStream;
import org.apache.hadoop.io.compress.CompressionOutputStream;

public class HdfsWrite {

  public static void main(String[] args) throws Exception {
    System.setProperty("java.library.path", "/home/redcreen/hadoop-current/lib/native/Linux-amd64-64");
    Field fieldSysPath = ClassLoader.class.getDeclaredField( "sys_paths" );
    fieldSysPath.setAccessible( true );
    fieldSysPath.set( null, null );
    
    
    Configuration conf = new Configuration();
    conf.set("fs.default.name", "hdfs://127.0.0.1:9000");
    
    conf.set(
        "io.compression.codecs",
        "org.apache.hadoop.io.compress.DefaultCodec,org.apache.hadoop.io.compress.LzoCodec");
    conf.set("io.compression.codec.lzo.class",
        "org.apache.hadoop.io.compress.LzoCodec");
    CompressionCodecFactory codecFactory = new CompressionCodecFactory(conf);
    
    FileSystem fs = FileSystem.get(conf);

    Path path = new Path("h1.lzo_deflate");
    CompressionCodec codec = codecFactory.getCodec(path);
    
    OutputStream outputStream = fs.create(path,true);

    CompressionOutputStream compressedOutput =
        codec.createOutputStream(outputStream);
    DataOutputStream out = new DataOutputStream(compressedOutput);
    out.writeChar('b');
    out.close();
    
    InputStream inputstream = fs.open(path);
    CompressionInputStream compressinput = codec.createInputStream(inputstream);
    DataInputStream din = new DataInputStream(compressinput);
    char ret = din.readChar();
    System.out.println("ret:"+ret );
    din.close();
  }
}

```

参考文章：
[Setting "java.library.path" programmatically][Setting "java.library.path" programmatically]
[Setting "java.library.path" programmatically]: http://blog.cedarsoft.com/2010/11/setting-java-library-path-programmatically/