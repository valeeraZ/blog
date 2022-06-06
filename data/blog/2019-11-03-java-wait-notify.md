---
title: Java并发-wait和notify
summary: wait() & notify() - 本文介绍线程状态，以及如何使用wait和notify来实现生产者消费者模型。
date: 2019-11-03
tags: ['Java']
---
本文介绍线程状态，以及如何使用`wait`和`notify`来实现生产者消费者模型。

# 关于线程的状态
java thread有五种状态类型

- 新建状态（New）：新创建了一个线程对象。
- 就绪状态（Runnable）：线程对象创建后，其他线程调用了该对象的start()方法。该状态的线程位于可运行线程池中，变得可运行，等待获取CPU的使用权。
- 运行状态（Running）：就绪状态的线程获取了CPU，执行程序代码。
- 阻塞状态（Blocked）：塞状态是线程因为某种原因放弃CPU使用权，暂时停止运行。直到线程进入就绪状态，才有机会转到运行状态。
- 死亡状态（Dead）：线程执行完了或者因异常退出了run()方法，该线程结束生命周期。  *

其中阻塞又可能是由以下几种情况造成：

1. 调用 sleep(毫秒数)，使线程进入“睡眠”状态。在规定的时间内，这个线程是不会运行的。
2. 用 suspend()暂停了线程的执行。除非线程收到 resume()消息，否则不会返回“可运行”状态。
3. 用 wait()暂停了线程的执行。除非线程收到 nofify()或者 notifyAll()消息，否则不会变成“可运行“。
4. 线程正在等候一些 IO（输入输出）操作完成。
5. 线程试图调用另一个对象的“同步”方法，但那个对象处于锁定状态，暂时无法使用。

当我们调用线程类的`sleep()`、`suspend()`、`yield()`、`wait()`等方法时会导致线程进入阻塞状态。

# 关于wait()和notify()
wait(): 调用任何对象的wait()方法会让当前线程进入等待，直到另一个线程调用同一个对象的notify()或notifyAll()方法。
notify():唤醒因调用这个对象wait()方法而阻塞的线程。  
首先，`sleep()`、`suspend()`、`yield()`等方法都隶属于 Thread 类，但`wait()/notify()`这一对却直接隶属于Object 类，也就是说，所有对象都拥有这一对方法。初看起来这十分不可思议，但是实际上却是很自然的，因为这一对方法阻塞时要释放占用的锁，而锁是任何对象都具有的，调用对象的 wait() 方法导致线程阻塞，并且该对象上的锁被释放。而调用对象的notify()方法则导致因调用该对象的 wait() 方法而阻塞的线程中随机选择的一个解除阻塞（但要等到获得锁后才真正可执行）。

其次，前面叙述的所有方法都可在任何位置调用，但是这一对方法却**必须在`synchronized`方法或块中调用**，理由也很简单，只有在synchronized 方法或块中当前线程才占有锁，才有锁可以释放。同样的道理，调用这一对方法的对象上的锁必须为当前线程所拥有，这样才有锁可以释放。因此，这一对方法调用必须放置在这样的 synchronized方法或块中，该方法或块的上锁对象就是调用这一对方法的对象。若不满足这一条件，则程序虽然仍能编译，但在运行时会出现IllegalMonitorStateException异常。

最后，关于 wait() 和 notify() 方法再说明三点：

1. 调用 notify() 方法导致解除阻塞的线程是从因调用该对象的 wait() 方法而阻塞的线程中随机选取的，我们无法预料哪一个线程将会被选择，所以编程时要特别小心，避免因这种不确定性而产生问题
2. 除了 notify()，还有一个方法 notifyAll() 也可起到类似作用，唯一的区别在于，调用 notifyAll() 方法将把因调用该对象的 wait() 方法而阻塞的所有线程一次性全部解除阻塞。当然，只有获得锁的那一个线程才能进入可执行状态。
3. wait()和notify()必须成对存在。

# 生产者消费者模型

箱子大小为5，生产者向其中放入消息，当箱子满时进入阻塞状态；消费者从其中取出消息，当箱子空时进入阻塞状态；通过notify唤醒对方。

生产者
```java
public class Producer extends Thread {
    public static final int MAX_BOX_SIZE = 5;
    Vector<String> messageBox = new Vector<>();

    @Override
    public void run() {
        try {
            while (true) {
                putMessage();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public synchronized void putMessage() throws InterruptedException {
        while (messageBox.size() == MAX_BOX_SIZE) {
            this.wait();//当箱子满后则进入等待
        }
        messageBox.add(new Date().toString());
        System.out.println("放入一条消息"+new Date().toString());
        this.notify();//放入消息后唤醒被锁住的线程（取消息线程）
    }

    public synchronized void getMessage() throws InterruptedException {
        while (messageBox.size() == 0) {
            this.wait();//当箱子空后进入等待
        }
        String message = (String) messageBox.firstElement();
        messageBox.removeElement(message);
        System.out.println("取出一条消息"+message);
        this.notify();//删除消息后唤醒被锁住的线程（放消息线程）
    }
}
```
消费者
```java
public class Consumer extends Thread{
    Producer producer;
    public Consumer(Producer producer){
        this.producer=producer;
    }
    @Override
    public void run() {
        try {
            while (true) {
                producer.getMessage();
                Thread.sleep(500);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        Producer producer=new Producer();
        Consumer consumer=new Consumer(producer);
        producer.start();
        consumer.start();
    }
}
```
---
版权声明：本文为CSDN博主「Heaven-Wang」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/suifeng3051/article/details/51852526