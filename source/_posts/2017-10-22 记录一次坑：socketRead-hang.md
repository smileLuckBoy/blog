---
title: 记录一次坑：socketRead hang
toc: true
date: 2017-10-22 21:10:36
tags:
categories:
---

最近在项目中，用到了大量的多线程中使用httpClient的场景，上线后发现线程池线程数量越来越少,通过jstack导出堆栈信息，发现大量的如下信息：
<!-- more -->
```
"pool-1034-thread-2" #7057672 prio=5 os_prio=0 tid=0x00007fbad81e9000 nid=0x711 runnable [0x00007fba579b9000]
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
    at java.net.SocketInputStream.read(SocketInputStream.java:170)
    at java.net.SocketInputStream.read(SocketInputStream.java:141)
    at sun.security.ssl.InputRecord.readFully(InputRecord.java:465)
    at sun.security.ssl.InputRecord.read(InputRecord.java:503)
    at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:973)
    - locked <0x00000000c7ccd168> (a java.lang.Object)
    at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1375)
    - locked <0x00000000c7ccd128> (a java.lang.Object)
    at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1403)
    at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1387)
    at org.apache.http.conn.ssl.SSLConnectionSocketFactory.createLayeredSocket(SSLConnectionSocketFactory.java:394)
    at org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.upgrade(DefaultHttpClientConnectionOperator.java:192)
    at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.upgrade(PoolingHttpClientConnectionManager.java:369)
    at org.apache.http.impl.execchain.MainClientExec.establishRoute(MainClientExec.java:415)
    at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:236)
    at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:184)
    at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:88)
    at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
    at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:184)
    at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:82)
    at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
    - <0x00000000c7c9e380> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```
看样子是httpClent读取资源时卡在了socketRead，检查代码后发现，socketReadTimeout是设置过了的，代码如下：

```java
RequestConfig.Builder reqConfigBuilder = RequestConfig.custom()
    // 从connection manager中获取连接的超时时间
    .setConnectionRequestTimeout(10000)
    // 请求获取数据的超时时间,即数据传输时间
    .setConnectTimeout(10000)
    // 建立连接的超时时间 
    .setSocketTimeout(10000)
RequestConfig reqConfig = reqConfigBuilder.build();
clientBuilder.setDefaultRequestConfig(reqConfig);
```

通过简单的demo测试，发现上述设置超时时间是有用的，只能说明在某些场景下，上述代码存在缺陷.
后续发现Stack Overflow中也有很多类似的问题，对于https协议，需要多设置一个PoolingHttpClientConnectionManager的超时参数：

```java
// SSL握手超时时间
manager.setDefaultSocketConfig(SocketConfig.custom().setSoTimeout(10000).build());
```
感谢[https calls ignore http.socket.timeout during SSL Handshake](https://issues.apache.org/jira/browse/HTTPCLIENT-1478)以及[httpClient4.3.5超时设置](http://zhoujinhuang.iteye.com/blog/2109067)。
