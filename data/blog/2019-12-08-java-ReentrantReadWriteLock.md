---
title: Java并发-ReentrantReadWriteLock应用
summary: Reader & Writer，Mutual Exclusion & Termination - 本文介绍经典的消费者-生产者模型，但这次用`ReentrantReadWriteLock`来实现并发时的线程安全。
date: 2019-12-08
tags: ['Java']
---

本文介绍经典的消费者-生产者模型，但这次用`ReentrantReadWriteLock`来实现并发时的线程安全。

三个主要的class

- Table，拥有数组`destination[sizeTable]`和`message[sizeTable]`

  - 当Writer写入消息时，对Table的destination数组遍历:   `for(i=0;i<sizeTable;i++)`

    - 若`destination[i]==-1`，即表示这个格字内无消息，此时可存放消息，方式为
      - `destination[i] = idReader`表示消息目的地 
      - `message[i] = msg`表示消息内容
  - 若遍历结束后，所有`destination[i]!=-1`即表示当前没有空出的格子，那此时线程让出锁，进入`await()`，直到有Reader读取到消息，释放一个格子
  
- Writer, 他使用table类中函数`writeMessage(int destination, int msg)`来写入消息。每个Writer写三个消息（每次写消息时，目标Reader和消息都是随机生成的）。也就是说，如果我们拥有4个Writer，每个Writer写入3个消息后线程结束，在成功写入4 * 3 = 12个消息后，我们可以说所有Writer线程都结束。

- Reader，他使用一个`while`循环不间断地尝试读取属于他(即`destination[i] == Myid`)的消息。如遍历过table后未找到，则抛出`NotFoundException`异常，然后再重新遍历。当成功读取到一条消息时，释放存放这个消息的格子，并`signalAll()`所有等待写入消息的Writer线程。

  当以下两个条件均成立时，我们可以断定这个Reader再也不会读取到任何消息了，此时这个Reader线程将结束。
  - table为空
  - 无未结束的Writer线程

# Table

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class table {
    private int mySize;
    private int destination[];
    private int message[];
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock(true);
    private final Condition write  = rwl.writeLock().newCondition();

    table(int mySize){
        this.mySize = mySize;
        this.destination = new int[mySize];
        this.message = new int[mySize];
        for(int i=0;i<mySize;i++){
            destination[i] = -1;//pas destination cible
        }
    }

    public void trace(String m){
        System.out.println("Table "+m);
        Thread.yield();
    }

    public boolean isEmpty(){
        boolean value = true;
        rwl.readLock().lock();
        try{
            for(int i = 0; i<mySize;i++){
                if(destination[i] != -1){
                    value = false;
                    break;
                }
            }
        }finally {
            rwl.readLock().unlock();
        }
        return value;
    }

    public void writeMessage(int msg, int dest) throws FullException {
        boolean restart = true;
        rwl.writeLock().lock();
        try {
            while (restart) {
                boolean full = true;
                for (int i = 0; i < mySize; i++) {
                    if (destination[i] == -1) {
                        full = false;
                        //successfully write a message, no restart
                        restart = false;
                        destination[i] = dest;
                        message[i] = msg;
                        break;
                    }
                }
                if (full) {
                    this.trace("is full, a writer is waiting");
                    // yield the lock and wait until the table is not full
                    write.await();
                }
            }
        }catch (InterruptedException e){
            System.err.println("An exception occurred.");
            throw new FullException("Wait for too long time");

        }finally {
            rwl.writeLock().unlock();
        }
    }

    public int readMessage(int id) throws NotFoundException {
        int value = 0;
        int i;
        rwl.readLock().lock();
        try {
            for (i = 0; i < mySize; i++) {
                if (destination[i] == id) {
                    value = message[i];
                    break;
                }
            }
            //a message has been read, a place in table is free
            if (i < mySize) {
                destination[i] = -1;//reset
                //as a place is free, try to wake up a waiting writer to write a message in it
                rwl.readLock().unlock();
                //lock the write (wait/notify)
                rwl.writeLock().lock();
                write.signalAll();
                rwl.writeLock().unlock();
                //get the read lock again to execute
                rwl.readLock().lock();
            } else {
                throw new NotFoundException("No message for reader " + id);
            }
        }finally {
            rwl.readLock().unlock();
        }
        return value;
    }
}
```

# Writer

```java
import java.util.Random;

public class AWriter implements Runnable {
    private int myID;
    private int nbReaders;
    private final table tn;
    private Random gen = new Random();
    //counter
    private static int cpt = 1;
    //Mutual exclusion
    private final static Object mutex = new Object();
    //for counting the active writers (who are not terminated)
    private static int activeWriters = 0;

    AWriter(table tab, int nbReaders){
        synchronized (mutex){
            myID = cpt++;
            activeWriters++;
        }
        this.tn = tab;
        this.nbReaders = nbReaders;
    }

    /**
     * all writers are terminated or not
     * @return true if all is terminated
     */
    public static boolean writersTerminated(){
        boolean value = false;
        synchronized (mutex){
            if(activeWriters == 0){
                value = true;
            }
        }
        return value;
    }

    public void trace(String m){
        System.out.println("Writer "+myID+" : "+m);
        Thread.yield();
    }

    public void waiting(){
        try{
            Thread.sleep(1000);
        }catch(Exception e){
            System.out.println(e);
        }
    }

    public void run(){
        boolean retry;
        this.trace("initialize");
        for(int i = 1; i <= 3; i++){
            int dest = gen.nextInt(nbReaders)+1;
            int msg = gen.nextInt(1000);
            this.trace("send "+msg+" to Reader "+dest+", message N° "+i);
            retry = true;
            while(retry){
                try{
                    // loop for writing a message
                    tn.writeMessage(msg,dest);
                    retry = false;
                }catch (Exception e){
                    //Possibly, the writer waits too long time for a place to be free
                    //It should retry
                    this.trace("I didn't make it write a message.");
                    retry = true;
                }
                this.waiting();
            }
        }
        this.trace("is terminated");
        synchronized (mutex){
            activeWriters--;
        }
    }

}
```

# Reader

```java
public class AReader implements Runnable {
    private int myID;
    private table tb;
    private static int cpt = 1;
    private int nbWriters;
    private static Object mutex = new Object();

    AReader(table tab,int nbWriters){
        synchronized (mutex){
            myID = cpt++;
        }
        this.tb = tab;
        this.nbWriters = nbWriters;
    }

    public void trace(String m){
        System.out.println("Reader "+myID+" : "+m);
        Thread.yield();
    }

    private void waiting(){
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void run(){
        this.trace("Initialize");
        //if there is/are still active writer(s)
        while (!AWriter.writersTerminated() || !tb.isEmpty()){
            try {
                int message = tb.readMessage(myID);
                this.trace("Message Received: "+ message);
            } catch (NotFoundException e) {
                this.trace(""+e);
            }
            this.waiting();
        }
    }
}
```

# 异常类

```java
public class FullException extends Exception{
    FullException(String m){
        super(m);
    }
}
public class NotFoundException extends Exception {
    NotFoundException(String m){
        super(m);
    }
}
```

# 测试

```java
public class test {
    public static void main(String args[]){
        int nbReaders = 5;
        int nbWriters = 3;
        int sizeTable = 3;
        table table = new table(sizeTable);
        for(int i = 0;i<nbWriters;i++){
            new Thread(new AWriter(table,nbReaders)).start();
        }

        for(int i=0;i<nbReaders;i++){
            new Thread(new AReader(table,nbWriters)).start();
        }
    }
}
```

