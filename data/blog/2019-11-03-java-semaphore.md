---
title: Java并发-Semaphore
summary: 信号量控制
date: 2019-11-03
tags: ['Java']
---
Semaphore实现为一种基于**计数**的信号量，Semaphore管理着一组虚拟的许可集合，这种许可可以作为某种凭证，来管理资源，在一些资源有限的场景下很有实用性，比如数据库连接，应用可初始化一组数据库连接，然后通过使用Semaphore来管理获取连接的许可，任何线程想要获得一个连接必须**首先获得一个许可**，然后再凭这个许可获得一个连接，这个许可将持续到这个线程归还了连接。在使用上，任何一个线程都需要通过`acquire`来获得一个Semaphore许可，这个操作可能会阻塞线程直到成功获得一个许可，因为资源是有限的，所以许可也是有限的，没有获得资源就需要阻塞等待其他线程**归还**Semaphore，而归还Semaphore操作通过`release`方法来进行，release会唤醒一个等待在Semaphore上的一个线程来尝试获得许可。如果想要达到一种**互斥**的效果，比如任何时刻只能有一个线程获得许可，那么就可以初始化Semaphore的数量为**1**，一个线程获得这个Semaphore之后，任何到来的通过acquire来尝试获得许可的线程都会被阻塞，直到这个持有Semaphore的线程调用了release方法来释放Semaphore。


# 使用
## 构造方法
```java
Semaphore(int permits)
Semaphore(int permits, boolean fair)
```
有两个构造方法，第一个参数表示可以同时访问资源的最大任务数，第二个是boolean类型，表示是否公平锁。

> 公平锁和非公平锁：程序在执行并发任务的时候，拿到同步锁的任务执行代码，其他任务阻塞等待，一旦同步锁被释放，CPU会正在等待的任务分配资源，获取同步锁。在这里又两种策略，CPU默认从等待的任务中**随机**分配，这是**非公平**锁；公平锁是按照**等待时间优先级**来分配，等待的时间越久，先获取任务锁。其内部是一个同步列队实现的。

## 主要方法
```java
acquire()//获取许可，Semephore任务数加一
release()//释放许可，Semephore任务数减一
```
还有几个方法如`tryAcquire()`尝试获取许可，返回boolean值，不阻塞;`availablePermits()`获取剩余任务许可数量，等等几个方法和Lock类的用法相似。详细使用参阅[api](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Semaphore.html)。  

简单的使用代码如下：
```java
// 创建一个计数阈值为5的信号量对象
// 只能5个线程同时访问
Semaphore semp = new Semaphore(5);

try {
    // 申请许可
    semp.acquire();
    try {
        // 业务逻辑
    } catch (Exception e) {

    } finally {
        // 释放许可
        semp.release();
    }
} catch (InterruptedException e) {

}
```
## 实例：线程池
```java
import java.util.concurrent.ExecutorService;  
import java.util.concurrent.Executors;  
import java.util.concurrent.Semaphore;  
public class SemaphoreTest{  
    public static void main(String[] args) {  
    //采用新特性来启动和管理线程——内部使用线程池  
        ExecutorService exec = Executors.newCachedThreadPool();  
        //只允许5个线程同时访问  
        final Semaphore semp = new Semaphore(5);  
        //模拟10个客户端访问  
        for (int index = 0; index < 10; index++){  
            final int num = index;  
            Runnable run = new Runnable() {  
                public void run() {  
                    try {  
                        //获取许可  
                        semp.acquire();  
                        System.out.println("线程" +   
                            Thread.currentThread().getName() + "获得许可："  + num);  
                        //模拟耗时的任务  
                        for (int i = 0; i < 999999; i++) ;  
                        //释放许可  
                        semp.release();  
                        System.out.println("线程" +   
                            Thread.currentThread().getName() + "释放许可："  + num);  
                        System.out.println("当前允许进入的任务个数：" +  
                            semp.availablePermits());  
                    }catch(InterruptedException e){  
                        e.printStackTrace();  
                    }  
                }  
            };  
            exec.execute(run);  
        }  
        //关闭线程池  
        exec.shutdown();  
    }  
}  
```
---
**版权声明**
本文部分来源于https://www.jianshu.com/p/f50455ad3514

