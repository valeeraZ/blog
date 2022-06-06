---
title: Java并发-服务器客户端多线程
summary: Multi-Thread Server-Client & usage of ArrayBlockingQueue 通过多线程模拟服务器-客户端发送消息
date: 2019-12-25
tags: ['Java']
---

通过多线程模拟服务器-客户端发送消息，其中

- 使用`ArrayBlockingQueue`储存客户端发送的消息

  - 客户端通过`put(query)`方法向其中放入消息
  - 服务器通过`take()`方法取出消息

  由于`ArrayBlockingQueue`内置了`ReentrantLock`锁机制，自动帮助我们处理队列满或空时无法放入或取出消息时的阻塞。简单的API如下：  

  | 方法 | 抛出异常  | 返回特殊值 | 阻塞   | 超时退出                 |
  | ---- | --------- | ---------- | ------ | ------------------------ |
  | 插入 | add(e)    | offer(e)   | put(e) | offer(e, time, timeunit) |
  | 移除 | remove()  | poll()     | take() | poll(time, timeunit)     |
  | 检查 | element() | peek()     | /      | /                        |

  - 抛出异常：是指当阻塞队列满时候，再往队列里插入元素，会抛出IllegalStateException(“Queue full”)异常。当队列为空时，从队列里获取元素时会抛出NoSuchElementException异常 。
  - 返回特殊值：插入方法会返回是否成功，成功则返回true。移除方法，则是从队列里拿出一个元素，如果没有则返回null
  - 一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到拿到数据，或者响应中断退出。当队列空时，消费者线程试图从队列里take元素，队列也会阻塞消费者线程，直到队列可用。
  - 超时退出：当阻塞队列满时，队列会阻塞生产者线程一段时间，如果超过一定的时间，生产者线程就会退出。

- 当服务器收到新的query时，新建一个新的子线程servant, 调用它处理线程，以避免服务器长时处理query阻塞。

# query类

```java
public class Query {
    Client sender;
    int value;
    Query(Client c, int v){
        this.sender = c;
        this.value = v;
    }
    public String toString(){
        return "("+sender+", "+value+")";
    }

}
```

# 客户端

服务器默认在`while(true)`一个死循环中运行，当所有的客户端均完成消息发送-得到回执后，最后一个客户端将服务器进程`interrupt`结束掉。

```java
import java.util.Random;
import java.util.concurrent.ArrayBlockingQueue;

public class Client implements Runnable {
    private int id;
    private ArrayBlockingQueue queue;
    private Random gen = new Random();
    private final Object alarm = new Object();
    private static int counter = 1;
    private static final Object mutex = new Object();
    private static int activeClient = 0;
    private Thread server;

    Client(ArrayBlockingQueue abq, Thread s){
        this.queue = abq;
        this.server = s;
        synchronized (mutex){
            id = counter++;
            activeClient++;
        }

    }

    private void trace(String m){
        System.out.println("Client "+id+": "+m);
    }

    public void solveQuery(){
        synchronized (alarm){
            alarm.notifyAll();
        }
    }

    public String toString(){
        return "Client "+id;
    }

    public void run(){
        this.trace("initialize");
        Query q = new Query(this, gen.nextInt(1000));
        this.trace("I send the query.");
        try {
            queue.put(q);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.trace("I wait for the response of server");
        try{
            synchronized (alarm){
                alarm.wait();
            }
        }catch (InterruptedException e){
            System.err.println("This should not happen: "+e);
        }
        this.trace("The query is resolved");
        synchronized (mutex){
            activeClient--;
            if(activeClient == 0){
                this.trace("Last client has been terminated, I kill the thread of Server.");
                server.interrupt();
            }
        }
    }
}
```

# 服务器端

当从`ArrayBlockingQueue`中取到消息时，构造一个新的线程servant用他来处理消息。

```java
import java.util.concurrent.ArrayBlockingQueue;

public class Server implements Runnable{
    private ArrayBlockingQueue<Query> queue;

    Server(ArrayBlockingQueue abq){
        this.queue = abq;
    }

    private void trace(String m){
        System.out.println("Server: "+m);
        Thread.yield();
    }

    public void run(){
        try{
            Query q;
            this.trace("initialize.");
            while (true){
                q = queue.take();
                this.trace("I resolve a query.");
                new Thread(new Servant(q)).start();
            }
        }catch (Exception e){
            this.trace("The last client terminates the server.");
        }

    }
}
```

## 服务器的子类--佣人

```java
public class Servant implements Runnable {
    private Query query;

    Servant(Query q){
        this.query = q;
    }

    private void trace(String m){
        System.out.println("Servant "+query+" : "+m);
    }

    public void run(){
        this.trace("initialize.");
        try{
            Thread.sleep(query.value);
        }catch (InterruptedException e){
            System.err.println("Servant This should not happen: "+e);
        }
        this.trace("I inform the server");
        query.sender.solveQuery();
    }
}
```

# 测试

```java
import java.util.concurrent.ArrayBlockingQueue;

public class test {
    static int sizeQueue;
    static int nbClient;

    public static void main(String args[]){
        try{
            sizeQueue = Integer.parseInt(args[0]);
            nbClient = Integer.parseInt(args[1]);
        }catch (ArrayIndexOutOfBoundsException e){
            System.err.println("Need for arguments: size of file and number of clients");
            return;
        }
        ArrayBlockingQueue queue = new ArrayBlockingQueue<Query>(sizeQueue,true);
        Thread server = new Thread(new Server(queue));
        for(int i=0;i<nbClient;i++){
            new Thread(new Client(queue, server)).start();
        }
        server.start();
    }

}
```
# 测试结果

sizequeue = 2, nbClient=3  
Client 1: initialize  
Client 2: initialize  
Server: initialize.  
Client 1: I send the query.  
Client 2: I send the query.  
Client 1: I wait for the response of server  
Client 2: I wait for the response of server  
Server: I resolve a query.  
Client 3: initialize  
Client 3: I send the query.  
Client 3: I wait for the response of server  
Server: I resolve a query.  
Server: I resolve a query.  
Servant (Client 1, 60) : initialize.  
Servant (Client 2, 335) : initialize.  
Servant (Client 3, 720) : initialize.  
Servant (Client 1, 60) : I inform the server  
Client 1: The query is resolved  
Servant (Client 2, 335) : I inform the server  
Client 2: The query is resolved  
Servant (Client 3, 720) : I inform the server  
Client 3: The query is resolved  
Client 3: Last client has been terminated, I kill the thread of Server.  
Server: The last client terminates the server.  

Process finished with exit code 0  

---

  参考链接： 

  [Java阻塞队列ArrayBlockingQueue使用及原理分析](https://blog.csdn.net/csdn_xpw/article/details/78400225)