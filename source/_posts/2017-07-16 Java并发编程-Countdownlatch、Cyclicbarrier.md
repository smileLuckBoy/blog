---
title: Java并发编程-Countdownlatch、Cyclicbarrier
toc: true
date: 2017-07-16 09:39:58
tags: Java并发编程
categories: JAVA
---

近期研究了下并发编程中的一些小技巧，涉及**Countdownlatch**、**Cyclicbarrier**，下面总结下这两个类的基本用法。

<!-- more -->

### Countdownlatch
#### 场景
包工头带领了四个小工生产零件，只有当四个小工将各自手中的活全部完成后，包工头才能开始向上级汇报工作。在这个场景中，就很适合用CountdownLatch
#### API
```java
//构造函数，其中参数count为计数器
public CountDownLatch(int count) { } 
//调用线程挂起等待
public void await() throws InterruptedException { }
//调用线程挂起等待timeout时间，如果超过timeout时间，调用线程开始继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { } 
//计数器减1
public void countDown() { } 
```
#### 场景解决方案
```java
public class CountDownLatchTest {

    public static class Worker implements Runnable {
        CountDownLatch countDownLatch;

        Worker(CountDownLatch countDownLatch) {
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(5000);
                System.out.println("Worker: " + Thread.currentThread().getName() + " finish the job!");
            } catch (Exception e) {

            } finally {
                //计数器减1
                countDownLatch.countDown();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        int workerNum = 4;
        CountDownLatch countDownLatch = new CountDownLatch(workerNum);
        for (int i = 0; i < workerNum; i++) {
            new Thread(new Worker(countDownLatch)).start();
        }
        //主线程等待紫线程执行完毕
        countDownLatch.await();
        //开始汇报工作
        System.out.println("all worker has finished！manager start to report the job!");
    }
}
```

代码输出如下：
```
Worker: Thread-2 finish the job!
Worker: Thread-3 finish the job!
Worker: Thread-1 finish the job!
Worker: Thread-0 finish the job!
manager start report the job!
```

### Cyclicbarrier
Cyclicbarrier直译过来就是循环栅栏，很抽象，倒不如仍叫Cyclicbarrier
#### 场景：
体育课，1000米测试，当所有的小伙伴都完成测试后才能解散自由活动，这个场景就很适合用Cyclicbarrier
#### API
```java
// 参数parties表示有多少个任务等待至barrier状态
public CyclicBarrier(int parties) { }
// 参数barrierAction表示当所有的任务都处于barrier状态时，将要执行的任务
public CyclicBarrier(int parties, Runnable barrierAction) { }
// 当前任务等待其它任务至barrier
public int await() throws InterruptedException, BrokenBarrierException { }
// 当前任务等待超过timeout就不再等待了
public int await(long timeout, TimeUnit unit) throws InterruptedException, BrokenBarrierException, TimeoutException { }
```
#### 场景解决方案
```java
public class CyclicBarrierTest {
    public static class Runner implements Runnable {
        CyclicBarrier cyclicBarrier;

        Runner(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(new Random().nextInt(5000));
                System.out.println("Runner " + Thread.currentThread().getName() + " has finished the 1000 race!");
                //等待其它小伙伴完成1000米测试
                cyclicBarrier.await();
            } catch (Exception e) {

            }
            //全部完成后开始自由活动
            System.out.println("Runner " + Thread.currentThread().getName() + " tart to do own things freely!");
        }
    }

    public static void main(String[] args) {
        int runnerCnt = 4;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(runnerCnt);
        for (int i = 0; i < runnerCnt; i++) {
            new Thread(new Runner(cyclicBarrier)).start();
        }
    }
}
```
代码输出如下：
```
Runner Thread-1 has finished the 1000 race!
Runner Thread-2 has finished the 1000 race!
Runner Thread-0 has finished the 1000 race!
Runner Thread-3 has finished the 1000 race!
Runner Thread-3 tart to do own things freely!
Runner Thread-1 tart to do own things freely!
Runner Thread-2 tart to do own things freely!
Runner Thread-0 tart to do own things freely!
```

### 总结
* 上文只是粗略的概述了上述两个Thread辅助类的用法，详细的还请参考源码
* 通过分析源码，不难发现这两个实现都是通过Lock以及Condition来实现的
* Countdownlatch侧重A线程组和B线程的关系
* Cyclicbarrier侧重线程组内线程的相互制约关系
* Cyclicbarrier是可以完成Countdownlatch类似的功能哦，想想它的构造函数哦

-- 未完待续