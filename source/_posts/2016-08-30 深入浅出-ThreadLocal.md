---
title: '深入浅出:ThreadLocal'
date: 2016-08-30 15:28:01
tags: [多线程,深入浅出]
categories: JAVA

toc: true
---


上次面试时，被面试官问到了有关ThreadLocal的内容，然后就没有然后了...

今天，整理一下学习ThreadLocal的内容，主要从适用场景、源码分析方面，和大家做一个分享，有不正确的地方希望大家指出来，一起进步~

<!-- more -->
## 场景

WEB应用中针对数据库大致有这么几种操作：

 * 创建数据库连接
 * 进行 DML 操作
 * 关闭数据库连接

如果有小A在执行DML操作时，突然小B关闭了数据库链接，那小A就不乐意了(握草，谁特么close了链接)


分析上述场景，我们可以推出原因 <font color="red">**数据库链接被不同线程共享了**</font> ，同时我们可以想到几个基本的解决方案



1. 将数据库连接变成同步的
2. 每次执行DML操作的时候都new一个数据库连接对象

分析上述解决方案

1. 执行简单的SQL居然先要等待获取数据库连接的使用权！
2. 频繁的创建、关闭数据库连接，不仅会使得程序执行性能下降，同时会加大服务器的压力！



<br>


## 初识ThreadLocal


>This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).


JDK是这样描述的，大体的意思就是ThreadLocal为每个线程存储了一些独享的变量，我们可以通过get和set方法来设置、获取这些变量的值


这里需要提前说明的是：虽然ThreadLocal直译过来是本地线程，但其本身并不是一个线程，它是通过操作线程(Thread)中的某个变量(ThreadLocalMap)来达到设置、获取当前线程变量副本的目的的


我们通过分析源码，先了解ThreadLocal内部的主要方法：


### protected T initialValue()

说明：
* 该方法返回副本的初始值
* 这个方法的修饰符是protected，很明显，这个是让我们在定义ThreadLocal变量的时候重写这个方法

```java
 protected T initialValue() {
    return null;
}
```

重写的示例很简单，如下：

```java
public static ThreadLocal<Integer> local = new ThreadLocal<Integer>(){
    @Override
    protected Integer initialValue() {
        return 0;
    }
};

```



### public void set(T value)
<font color="red">**在这个方法中我们将揭开为什么ThreadLocal可以为每个线程(Thread)提供独立的变量副本！**</font>

1.这里首先通过Thread.currentThread()获取当前线程(Thread)

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

2.随后通过getMap()方法，获取了当前线程的threadLocals值

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```


3.我们跟踪查看Thread类中的threadLocals，它是ThreadLocal内部类ThreadLocalMap的一个对象

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

4.通过map.set(this, value)设置变量副本值
ThreadLocalMap，数据结构是MAP&lt;K,V&gt;，K为<font color="red">当前ThreadLocal对象</font>，V就是变量副本
到这里其实已经很明白了，线程中的副本变量是存储在线程的ThreadLocal.ThreadLocalMap对象中的


大家可以思考下K值为什么当前ThreadLocal对象呢？或者我们换一种方式，如果一个线程中需要存储多个不同的变量副本该怎么办呢？


5.总结分析：
* ThreadLocalMap是Thread的内部属性(每个线程都不一样)，它存储了线程变量副本
* ThreadLocal是通过set、get方法来创建、操作ThreadLocalMap的(添加&lt;K,V&gt;)
* 使用当前线程的ThreadLocalMap的关键是将当前ThreadLocal的实例作为K进行存储

6.隔离机制:
* 纵向隔离 : 在不同线程中，每个线程访问的都是各自的ThreadLocalMap
* 横向隔离 : 同一个线程中，不同的ThreadLocal实例作为ThreadLocalMap的K值



### public T get()
说明：
* 从ThreadLocalMap中取变量副本的值，K值为当前ThreadLocal实例对象

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```


### public void remove()
说明：
* 移除当前线程中这个ThreadLocal中的变量副本
* 当移除后再次访问的话，即会走initialValue()方法来初始化
* 此方法不是必要的，当Thread结束时，ThreadLocal中的变量副本均会自动GC回收
* 手动执行该方法可以尽早释放占用的内存

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

<br>

## 实践应用

### 数据库连接
既然ThreadLocal可以为每个线程保存自己独享的一份变量副本，那么对于文章开始提到的场景，我们可以把DBConnection放置在ThreadLocal中，这样每个线程就有一个属于自己的数据库连接啦，这样既避免了同步，又避免了一个线程中多次创建、销毁DBConnection

伪代码如下，相信大家看下就懂了哈~

```java
public class ConnectionManager {  
  
    /** 线程内共享Connection，ThreadLocal通常是private static类型的 */  
    private static ThreadLocal<Connection> connection = new ThreadLocal<Connection>(){
    	@Override
        protected Connection initialValue() {
        	// 创建一个新的数据库连接
            return new Connection();
        }
    };
      
    
    /**
     * 获取当前数据库连接
     */
     public static Connection getConnection() {
     	return connection.get();
     }
      
    /** 
     * 关闭当前数据库连接 
     */  
    public static void close() {  
        // 获取当前线程内共享的Connection  
        Connection conn = ConnectionManager.getConnection();  
        try {  
            // 判断是否已经关闭  
            if(conn != null && !conn.isClosed()) {  
                // 关闭资源  
                conn.close();  
                // 移除Connection  
                connection.remove();  
                conn = null;  
            }  
        } catch (SQLException e) {  
            // 异常处理  
        }  
    }  
}
```


### DEMO
或者如果感觉上述代码无法运行的话，这里有个小demo，供大家测试~

```java
public class ThreadLocalTest {

    public static ThreadLocal<Integer> local = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };


    public static void main(String[] args) {
        Thread[] t = new Thread[5];

        for (int i = 0; i < t.length; i++) {
            t[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    Integer num = local.get();
                    // 这里只是对num做简单的++操作
                    local.set(++num);
                    // 如果num是共享的话,那么localNum的值不会都是1
                    System.out.println("ThreaName:" + Thread.currentThread().getName() + "   localNum:" + local.get());
                }
            });
        }

        for (Thread thread : t) {
            thread.start();
        }
    }
}
```

这里就不截图了，大家可以想想print的内容，然后运行程序比对一下~


<br>


