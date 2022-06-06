---
title: Java并发-进程，线程及synchronized关键字
summary: 线程安全 - 本文介绍并行与并发的区别，线程与进程的区别，以及使用`synchronized`关键字来保证线程安全。
date: 2019-11-02
tags: ['Java']
---
本文介绍并行与并发的区别，线程与进程的区别，以及使用`synchronized`关键字来保证线程安全。

# 并行与并发
并行性（parallel）：指在同一时刻，有多条指令在多个处理器上同时执行；  
并发性（concurrency）：指在同一时刻只能有一条指令执行，但多个进程指令被快速轮换执行，使得在宏观上具有多个进程同时执行的效果。  
我们在解决编程问题时，通常使用顺序编程来解决，即程序中的所有事物在任意时刻都只能执行一个步骤。然而对于某些问题，我们希望能够并行地执行程序中的多个部分，来达到我们想要的效果。在单处理器机器中，我们可以将程序划分为多个部分，然后每个部分由该处理器并发执行。在多处理器机器中，我们可以将程序划分多个部分，然后每个部分分别在多个处理器上并行执行。当然为了更加充分利用CPU资源，我们也可以在多个处理器上并发执行，那么在这我们就涉及到了另一种编程模式了并发编程。并发编程又叫多线程编程。并发编程使我们可以将程序划分为多个分离的、独立运行的任务。通过使用多线程机制，每个独立任务都将由线程来驱动。一个线程就是在进程中的一个单一的顺序控制流，单个进程可以拥有多个"并发执行"的任务。这样使程序的每个任务，都好像拥有一个自己的CPU一样。



# 线程与进程

线程(Thread)也被称作轻量级进程（Lightweight Process）。线程是进程的执行单元，就像进程在操作系统中的地位一样，线程在程序中是独立的、并发的执行流。当进程被初始化后，主线程就被创建了。对于绝大多数的应用程序来说，通常仅要求有一个主线程，但也可以在该进程内创建多条顺序执行流，这些顺序执行流就是线程，每个线程也是互相独立的。

线程是进程的组成部分，一个进程可以拥有多个线程，**一个线程必须有一个父进程**。线程可以拥有自己的堆栈、自己的程序计数器和自己的局部变量，但不拥有系统资源，它与父进程的其他线程共享该进程所拥有的全部资源。因为多个线程共享父进程里的全部资源，因此编程更加方便；但必须更加小心，我们必须确保线程不会妨碍同一进程里的其他线程。

线程可以完成一定的任务，可以与其他线程共享父进程中的**共享变量及部分环境**，相互之间协同来完成进程所要完成的任务。线程是**独立运行**的，它并不知道进程中是否还有其他线程存在，线程的执行是**抢占式**的，也就是说，当前运行的线程在任何时候都可能被挂起，以便另外一个线程可以运行。

一个线程可以创建和撤销另一个线程，同一个进程中的多个线程之间可以并发执行。从逻辑角度来看，多线程存在于一个应用程序中，让一个应用程序中可以有多个执行部分同时执行，但操作系统无须将多个线程看作多个独立的应用，对多线程实现调度和管理以及资源分配。线程的调度和管理由进程本身负责完成。

线程与进程的比较：
1. 进程之间不能共享内存，但线程之间共享内存非常容易
2. 系统创建进程时需要为该进程重新分配系统资源，但创建线程则代价小得多，因此使用多线程来实现多任务并发比多进程的效率高
3. Java语言内置了多线程功能支持，而不是单纯地作为底层操作系统的调度方式，从而简化了Java的多线程编程

# 线程安全

"非线程安全"问题存在于"实例变量"中，如果是方法内部的私有变量，则不存在"非线程安全"问题，所得结果也就是"线程安全"的了。  
如果两个线程同时操作对象中的实例变量，则会出现"非线程安全"，解决办法就是在方法前加上``synchronized``关键字即可。

# synchronized关键字

## 对象锁
synchronized取得的锁都是对象锁，而不是把一段代码或方法当做锁。哪个线程先执行带synchronized关键字的方法，则哪个线程就持有该方法所属对象的锁Lock，那么其他线程只能呈等待状态，前提是多个线程访问的是**同一个对象**。

- 修饰实例方法，作用于当前**对象实例**加锁，进入同步代码前要获得当前对象实例的锁
- 修饰静态方法，作用于当前**类对象**加锁，进入同步代码前要获得当前类对象的锁：也就是给当前**类**加锁，会作用于类的所有对象实例，因为静态成员不属于任何一个实例对象，是类成员(static 表明这是该类的一个静态资源，不管new了多少个对象，只有一份，所以对该类的所有对象都加了锁)。所以如果一个线程A调用一个实例对象的非静态 synchronized 方法，而线程B需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，因为**访问静态 synchronized 方法占用的锁是当前类的锁**，而访问非静态 synchronized 方法占用的锁是当前实例对象锁。
- 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。 和 synchronized 方法一样，`synchronized(this)`代码块也是锁定当前对象的。synchronized 关键字加到 static 静态方法和 synchronized(class)代码块上都是是给 Class 类上锁。这里再提一下：synchronized关键字加到非 static 静态方法上是给对象实例上锁。另外需要注意的是：尽量不要使用 synchronized(String a) 因为JVM中，字符串常量池具有缓冲功能！

因此，我们可以知道：
1. 当一个线程正在访问一个对象的synchronized方法，那么其他线程不能访问该对象的其他synchronized方法。这个原因很简单，因为一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他synchronized方法。
2. 当一个线程正在访问一个对象的synchronized方法，那么其他线程能访问该对象的非synchronized方法。这个原因很简单，访问非synchronized方法不需要获得该对象的锁，假如一个方法没用synchronized关键字修饰，说明它不会使用到临界资源，那么其他线程是可以访问这个方法的，
3. 如果一个线程A需要访问对象object1的synchronized方法fun1，另外一个线程B需要访问对象object2的synchronized方法fun1，即使object1和object2是同一类型），也不会产生线程安全问题，因为他们访问的是不同的对象，所以不存在互斥问题。

## 对方法synchronized
下面这段代码中两个线程分别调用`insertData`对象插入数据：  
```java
import java.util.*;
public class test_synchronized {
 
    public static void main(String[] args)  {
        final InsertData insertData = new InsertData();
         
        new Thread() {
            public void run() {
                insertData.insert(Thread.currentThread());
            };
        }.start();
         
         
        new Thread() {
            public void run() {
                insertData.insert(Thread.currentThread());
            };
        }.start();
    }  
}
 
class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();
     
    public void insert(Thread thread){
        for(int i=0;i<5;i++){
            System.out.println(thread.getName()+"在插入数据"+i);
            arrayList.add(i);
        }
    }
}
```
运行结果：  
![java_synchronized.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856521741_1341.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)  
说明两个线程在同时执行insert方法。

而如果在insert方法前面加上关键字synchronized的话：
```java
public synchronized void insert(Thread thread){
    for(int i=0;i<5;i++){
        System.out.println(thread.getName()+"在插入数据"+i);
        arrayList.add(i);
    }
}
```
运行结果为：  
![concurrence.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856522616_9897.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)  
从上输出结果说明，由于两个线程无法取得同一把对象锁，Thread-1插入数据是等Thread-0结束后才进行的。说明Thread-0和Thread-1是顺序执行insert方法的。

## 对代码块synchonized
synchronized代码块类似于以下这种形式：
```java
synchronized(synObject) {
         
    }
```
当在某个线程中执行这段代码块，该线程会获取对象synObject的锁，从而使得其他线程无法同时访问该代码块。**synObject可以是this，代表获取当前对象的锁，也可以是类中的一个属性，代表获取该属性的锁。**

比如上面的insert方法可以改成以下两种形式：

```java
class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();
     
    public void insert(Thread thread){
        synchronized (this) {
            for(int i=0;i<100;i++){
                System.out.println(thread.getName()+"在插入数据"+i);
                arrayList.add(i);
            }
        }
    }
}
 

class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();
    private Object object = new Object();
     
    public void insert(Thread thread){
        synchronized (object) {
            for(int i=0;i<100;i++){
                System.out.println(thread.getName()+"在插入数据"+i);
                arrayList.add(i);
            }
        }
    }
}
```
从上面可以看出，synchronized代码块使用起来比synchronized方法要灵活得多。因为也许一个方法中只有一部分代码只需要同步，如果此时对整个方法用synchronized进行同步，会影响程序执行效率。而使用synchronized代码块就可以避免这个问题，synchronized代码块可以实现只对需要同步的地方进行同步。

## 对静态方法

如果一个线程执行一个对象的非static synchronized方法，另外一个线程需要执行这个对象所属类的static synchronized方法，此时不会发生互斥现象，因为访问static synchronized方法占用的是类锁，而访问非static synchronized方法占用的是对象锁，所以不存在互斥现象。

```java
public class Test {
 
    public static void main(String[] args)  {
        final InsertData insertData = new InsertData();
        new Thread(){
            @Override
            public void run() {
                insertData.insert();
            }
        }.start(); 
        new Thread(){
            @Override
            public void run() {
                insertData.insert1();
            }
        }.start();
    }  
}
 
class InsertData { 
    public synchronized void insert(){
        System.out.println("执行insert");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("执行insert完毕");
    }
     
    public synchronized static void insert1() {
        System.out.println("执行insert1");
        System.out.println("执行insert1完毕");
    }
}
```

运行结果：

![synchronized_static.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856523613_8601.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0) 

第一个线程里面执行的是insert方法，不会导致第二个线程执行insert1方法发生阻塞现象。

---
**版权声明**  
文章代码部分出处：  
作者：Matrix海子  
出处：http://www.cnblogs.com/dolphin0520/