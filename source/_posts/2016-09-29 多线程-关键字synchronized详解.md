---
title: 多线程-关键字synchronized详解
toc: true
date: 2016-09-29 09:33:03
tags: thread
categories: JAVA
---

近期在看《JAVA 多线程编程核心技术》这本书，先说下读后感(好像读完书总要搞这么个读后感表明自己研读过)，标价69.00RMB，还是有点肉疼，不过总还是有点收获的(建议大家还是多读读各类博文，书就不建议购买了...)，下面我把自己理解的一些粗俗易懂的知识点整理下，没啥太深入的东西，大牛可忽略此文了，像我一样的JAVA小白可以参考参考~

<!-- more -->

### synchronized 介绍

synchronized是JAVA中的一个关键字，主要是用来解决多线程之间的 <font color="red">共享变量同步</font> 问题的，举个栗子：
A账户上有1000.00RMB，A的老婆和A的女儿刚好都 <font color="red">**同时**</font> 要提取600.00RMB（A真特么苦逼），老婆在取钱时，一看还有1000.00RMB，取走了600.00RMB，同时，在老婆取钱操作还没完成时，女儿一看账户上仍有1000.00RMB，也取走了600.00RMB，这样的话，A的账户就剩下-200.00RMB了，银行肯定不乐意的啊。
synchronized关键字呢就是解决这类问题的，大体的思路就是当A的老婆和女儿同时取钱时，谁先获得者个账户的使用权(锁,从更加专业的角度来说，应该称之为监视器**monitor**)，谁就可以先操作账户，另外一个人就得等待，这样的话，当一个人取完钱时，账户就只剩余400.00RMB了，另外一个最多只能取400了。

上述举到的例子中，A的账户就是一个被多个线程(Thread老婆 和 Thread女儿)同时操作的共享变量！


### synchronized 同步方法

#### 单个synchronized方法
代码原型如下：
```java
public class Service {
    synchronized public void A(){}
}
```

看到这样的代码，也许你会问： synchronized获得的是谁的monitor啊？这个问题其实也一度困扰了我好久，后来，我明白了他获取的是Service对象的monitor，我们来写段代码说明下吧：
```java
public class Account {

    private double accountMoney = 1000;

    synchronized public void subAccount(double money) {
    	// 取款操作不瞬间完成
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (money > accountMoney) {
            System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + this.accountMoney + ",余额不足请充值!");
            return;
        }
        this.accountMoney = accountMoney - money;
        System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + this.accountMoney);
    }
}

public class Run {
    Account account = new Account();
    Runnable run = new Runnable() {
        @Override
        public void run() {
            account.subAccount(600);
        }
    };
    public static void main(String[] args) {
        Run r = new Run();
        Thread[] threads = new Thread[2];
        for (int i = 0 ;i < threads.length;i++){
            threads[i] = new Thread(r.run);
            threads[i].setName(i % 2 == 0 ? "老婆" : "女儿");
        }

        for (int i = 0 ; i < threads.length;i++){
            threads[i].start();
        }
    }
}
```
代码输出如下：

```
老婆取款600.0当前账户余额400.0
女儿取款600.0当前账户余额400.0,余额不足请充值!

```
去掉Service中的synchronized，输出如下：
```
女儿取款600.0当前账户余额400.0
老婆取款600.0当前账户余额400.0
```

我们看到在main方法中，同时启动了女儿和老婆的取款线程，当不加 synchronized关键字时，呈现了均取出600的不正确结果，加上此关键字，结果是对的，下面我们分析下：

加上 synchronized关键字后：
老婆和女儿的取款线程首先要获取Account的对象account的monitor,当然，只会有一个会获取到（谁先获取这个全拼运气），另外一个会等待，当先获取到monitor的线程执行完毕后会释放Monitor，另外一个线程才会执行。
需要注意的是：这里只申明了一个Account的对象，如果申明两个，两个线程分别采用一个，想想会出现什么结果？<a href="#5F">想看结果请戳这里</a> 

#### 多个synchronized方法
话不多说，上代码
```java
public class Account {

    private double accountMoney = 1000;

	// 通过网银扣钱
    synchronized public void subAccountFromNet(double money) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (money > accountMoney) {
            System.out.println(Thread.currentThread().getName() + "通过网银取款" + money + "当前账户余额" + this.accountMoney + ",余额不足请充值!");
            return;
        }
        this.accountMoney = accountMoney - money;
        System.out.println(Thread.currentThread().getName() + "通过网银取款" + money + "当前账户余额" + this.accountMoney);
    }


	// 通过POS机刷卡
    synchronized public void subAccountFromCard(double money) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (money > accountMoney) {
            System.out.println(Thread.currentThread().getName() + "通过POS机取款" + money + "当前账户余额" + this.accountMoney + ",余额不足请充值!");
            return;
        }
        this.accountMoney = accountMoney - money;
        System.out.println(Thread.currentThread().getName() + "通过POS机取款" + money + "当前账户余额" + this.accountMoney);
    }
}


public class Run {
    Account service = new Account();
    Runnable net = new Runnable() {
        @Override
        public void run() {
            service.subAccountFromNet(600);
        }
    };

    Runnable pos = new Runnable() {
        @Override
        public void run() {
            service.subAccountFromCard(600);
        }
    };

    public static void main(String[] args) {
        Run r = new Run();

        Thread thread1 = new Thread(r.net,"老婆");
        Thread thread2 = new Thread(r.pos,"女儿");

        thread1.start();
        thread2.start();
    }
}

```

代码输出如下：
```
老婆通过网银取款600.0当前账户余额400.0
女儿通过POS机取款600.0当前账户余额400.0,余额不足请充值!
```

从这里其实我们可以看出，即使是同一个类中的不同的synchronized方法，当某一个线程有先获取了该对象的监视器并且执行了其中另外一个方法的时候，其它的线程此时是无法访问另外的synchronized方法的。

#### 静态synchronized方法
代码原型如下：
```java
public class Service {
    synchronized static public void A(){}
}
```
看起来就是在方法上加个static嘛，我们知道，static关键字是针对类的，并不是针对对象的，这里我们可以简单的将static synchronized方法理解为它要获取的是**类**的锁，测试代码如下：
<a id="5F"></href> 
```java
public class Account {

    private double accountMoney = 1000;

    synchronized public void subAccount(double money) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (money > accountMoney) {
            System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + accountMoney + ",余额不足请充值!");
            return;
        }
        accountMoney = accountMoney - money;
        System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + accountMoney);
    }

}

public class Run {
	//这里申明了两个Account对象
    Account service1 = new Account();
    Account service2 = new Account();
    Runnable net = new Runnable() {
        @Override
        public void run() {
            service1.subAccount(600);
        }
    };

    Runnable pos = new Runnable() {
        @Override
        public void run() {
            service2.subAccount(600);
        }
    };

    public static void main(String[] args) {
        Run r = new Run();

        Thread thread1 = new Thread(r.net,"老婆");
        Thread thread2 = new Thread(r.pos,"女儿");

        thread1.start();
        thread2.start();
    }
}

```
运行结果如下：
```
老婆取款600.0当前账户余额400.0
女儿取款600.0当前账户余额400.0
```

这里也就是我们之前提到的问题，如果我们申明两个Account对象，而且每个线程都使用不同的Account对象的结果，结果为什么会这样，很简单，因为每个对象都有属于自己的monitor！

如果我们就是需要申明不同的Account对象，并且还要求结果正确该怎么办呢？解决方案就是我们之前提到的static,在subAccount方法加上static关键字，试试呢？

运行结果如下：

```
老婆取款600.0当前账户余额400.0
女儿取款600.0当前账户余额400.0,余额不足请充值!
```
结果是正确的，因为static synchronized方法相当等于要获取Class的锁，Class锁对所有的类实例都是起作用的，也就是说这个类下的所有对象都使用的是同一把锁，但是Class锁和对象锁是不一样的！ 



#### 非synchronized方法
即使某个线程通过获取了对象的minotor，执行了synchronized方法，其他线程仍然可以同一时间访问这个对象的非synchronized方法，结论的认证大家自己写写demo就能明白


### synchronized 代码块

理解了synchronized同步方法，其实这个也大同小异，synchronized同步语句相比synchronized方法具有的优势就是一旦某个线程访问了一个synchronized方法，那么其它线程就不能访问这个对象的其它的synchronized方法，从时间上来说，造成了浪费，但是synchronized代码块可以有效的解决这个问题，只有在synchronized代码块中的代码才有可能造成阻塞，而外部的代码则不会造成阻塞与等待。


#### synchronized(Object)
代码原型如下：
```java
public void A(){
    synchronized (obj){
    }
}
```

刚才的例子同样可以用此方法实现
```java
public void subAccount(double money) {
    synchronized (this) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (money > accountMoney) {
            System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + this.accountMoney + ",余额不足请充值!");
            return;
        }
        this.accountMoney = accountMoney - money;
        System.out.println(Thread.currentThread().getName() + "取款" + money + "当前账户余额" + this.accountMoney);
    }

}
```
当然this可以换成其它的Object，不过需要注意的是
* 调用这两个方法的线程之间相互影响
方法一: synchronized(obj1){}
方法二: synchronized(ojb1){}

* 调用这两个方法的线程之间相互不影响
方法一: synchronized(obj1){}
方法二: synchronized(obj2){}



#### synchronized(Class)

代码原型如下：
```java
public void A(){
    synchronized (class){
    }
}
```
这个和static类型的synchronized方法是类似的，就不具体解释了。


### 总结

* 使用synchronized的关键在于你要清楚你获取的是哪个对象或者类的Monitor
* synchronized方法（代码块）和非synchronized方法就像你进了一个大杂院，有很多屋子，每间屋子只能容纳一人，有些屋子没有锁子，有些屋子上了锁，而且所有的锁都一样，这样，当第一个一个人进入大杂院，先拿到钥匙，并且进入一间有锁的屋子，另外一个人就无法进入这间或者另外一间带锁的屋子了，但是可以进入不带锁的屋子。


