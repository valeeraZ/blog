---
title: Java并发-ExecutorService使用
summary: 简介Executor 框架与ExecutorService
date: 2019-12-28
tags: ['Java']
---

在 Java 5 之后，并发编程引入了一堆新的启动、调度和管理线程的API。`Executor` 框架便是 Java 5 中引入的，其内部使用了线程池机制，它在 `java.util.cocurrent` 包下，通过该框架来控制线程的启动、执行和关闭，可以简化并发编程的操作。因此，在 Java 5之后，通过 `Executor` 来启动线程比使用 Thread 的 start 方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免 `this` 逃逸问题。

> 补充：`this` 逃逸是指在构造函数返回之前其他线程就持有该对象的引用. 调用尚未构造完全的对象的方法可能引发令人疑惑的错误。

# Executor框架

1. 任务(`Runnable` /`Callable`)
执行任务需要实现的 `Runnable` 接口 或 `Callable`接口。`Runnable` 接口或 `Callable` 接口 实现类都可以被 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor` 执行。

2. 任务的执行(`Executor`)
如下图所示，包括任务执行机制的核心接口 `Executor` ，以及继承自 `Executor` 接口的 `ExecutorService` 接口。`ThreadPoolExecutor` 和 `ScheduledThreadPoolExecutor` 这两个关键类实现了 `ExecutorService` 接口。

![任务的执行相关接口.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856540813_6568.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

3. 异步计算的结果(`Future`)
`Future` 接口以及 `Future` 接口的实现类 `FutureTask` 类都可以代表异步计算的结果。
当我们把 `Runnable`接口 或 `Callable` 接口 的实现类提交给 `ThreadPoolExecutor` 或 `ScheduledThreadPoolExecutor` 执行。（调用 `submit()` 方法时会返回一个 `FutureTask` 对象）

![Executor.jpeg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856541542_645.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

总结：  
1. 主线程首先要创建实现 `Runnable` 或者 `Callable` 接口的任务对象。
2. 把创建完成的实现 `Runnable`/`Callable`接口的对象直接交给 `ExecutorService` 执行: `ExecutorService.execute(Runnable command)`或者也可以提交给 `ExecutorService` 执行：`ExecutorService.submit(Runnable task)`或 `ExecutorService.submit(Callable <T> task)`。
3. 如果执行 `ExecutorService.submit(...)`，`ExecutorService` 将返回一个实现`Future`接口的对象（我们刚刚也提到过了执行 `execute()`方法和 `submit()`方法的区别，`submit()`会返回一个 `FutureTask` 对象）。由于 `FutureTask` 实现了 `Runnable`，我们也可以创建 `FutureTask`，然后直接交给 `ExecutorService` 执行。
4. 最后，主线程可以执行 `FutureTask.get()`方法来等待任务执行完成。主线程也可以执行 `FutureTask.cancel(boolean mayInterruptIfRunning)`来取消此任务的执行。

# ExecutorService简介

## 创建 ExecutorService

1. `ExecutorService executorService0 = Executors.newSingleThreadExecutor();` 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
    ```java
    ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
    for (int i = 0; i < 10; i++) {
        final int index = i;
        singleThreadExecutor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(index);
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        });
    }
    ```
    结果依次输出，相当于顺序执行各个任务。现行大多数GUI程序都是单线程的。Android中单线程可用于数据库操作，文件操作，应用批量安装，应用批量删除等不适合并发但可能IO阻塞性及影响UI线程响应的操作。

2. `ExecutorService executorService1 = Executors.newCachedThreadPool();` 创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线 程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。
    ```java
    ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
    for (int i = 0; i < 10; i++) {
        final int index = i;
        try {
            Thread.sleep(index * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        cachedThreadPool.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println(index);
            }
        });
    }
    ```
    线程池为无限大，当执行第二个任务时第一个任务已经完成，会复用执行第一个任务的线程，而不用每次新建线程。
3. `ExecutorService executorService2 = Executors.newFixedThreadPool(10); `创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
    ```java
    ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
    for (int i = 0; i < 10; i++) {
        final int index = i;
        fixedThreadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(index);
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        });
    }
    ```
    因为线程池大小为3，每个任务输出index后sleep 2秒，所以每两秒打印3个数字。定长线程池的大小最好根据系统资源进行设置。如Runtime.getRuntime().availableProcessors()。可参考PreloadDataCache。

4. `ExecutorService executorService3 = Executors.newScheduledThreadPool(10);` 创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。  
延迟执行：
    ```java
    ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
    scheduledThreadPool.schedule(new Runnable() {
        @Override
        public void run() {
            System.out.println("delay 3 seconds");
        }
    }, 3, TimeUnit.SECONDS);
    ```
    表示延迟3秒执行。  

    定期执行：
    ```java
    scheduledThreadPool.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            System.out.println("delay 1 seconds, and excute every 3 seconds");
        }
    }, 1, 3, TimeUnit.SECONDS);

    ```
    表示延迟1秒后每3秒执行一次。
	
## 使用ExecutorService

```java
execute(Runnable) 
submit(Runnable) 
submit(Callable) 
invokeAny() 
invokeAll() 
```

1. execute(Runnable)

   方法`execute(Runnable)`接收一个`java.lang.Runnable` 对象作为参数，并且以异步的方式执行它。

   ```java
   ExecutorService executorService = Executors.newSingleThreadExecutor();  
   executorService.execute(new Runnable() {  
       public void run() {  
           System.out.println("Asynchronous task");  
       }  
   });   
   executorService.shutdown();  
   ```

   使用这种方式没有办法获取执行 Runnable 之后的结果，如果你希望获取运行之后的返回值，就必须使用 接收 Callable 参数的 execute() 方法，后者将会在下文中提到。

2. submit(Runnable)

   方法 `submit(Runnable)` 同样接收一个Runnable 的实现作为参数，但是会返回一个`Future`对象。这个`Future`对象可以用于判断 Runnable是否结束执行。

   ```java
   ExecutorService executorService = Executors.newSingleThreadExecutor();
   Future future = executorService.submit(new Runnable() {  
       public void run() {  
           System.out.println("Asynchronous task");  
       }  
   });  
   //如果任务结束执行则返回 null  
   try{
       System.out.println("future.get()=" + future.get());  
   }catch(InterruptedException e){
       e.printStackTrace();
   }catch(ExecutionException e){
       e.printStackTrace();
   }
   executorService.shutdown();
   ```

3. submit(Callable)

   方法 `submit(Callable)` 和方法 `submit(Runnable)` 比较类似，但是区别则在于它们接收不同的参数类型。`Callable` 的实例与 `Runnable` 的实例很类似，但是 `Callable` 的 `call()` 方法可以返回结果。方法 `Runnable.run()` 则不能返回结果。

   `Callable` 的返回值可以从方法 `submit(Callable)` 返回的 `Future` 对象中获取。

   ```java
   ExecutorService executorService = Executors.newSingleThreadExecutor();
   Future future = executorService.submit(new Callable<Integer>() {  
       public Integer call() {  
           System.out.println("Asynchronous task");  
           return 100;
       }  
   });  
   //如果任务结束执行则返回100
   try{
       System.out.println("future.get()=" + future.get());  
   }catch(InterruptedException e){
       e.printStackTrace();
   }catch(ExecutionException e){
       e.printStackTrace();
   }
   executorService.shutdown();
   ```
   
4. invokeAny()

   方法`invokeAny()`接收一个包含`Callable`对象的**集合**作为参数，调用该方法不会返回 Future 对象，而是返回集合中某个 `Callable` 对象的结果，而且无法保证调用之后返回的结果是哪个 `Callable`，只知道它是这些 `Callable` 中一个执行结束的 `Callable` 对象。
   如果一个任务运行完毕或者抛出异常，方法会取消其它的 Callable 的执行。

   ```java
   Set<Callable<String>> callables = new HashSet<Callable<String>>();      
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 1";  
       }  
   });  
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 2";  
       }  
   });  
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 3";  
       }  
   });
   String result = "";  
   try {
       result = executorService.invokeAny(callables);  
   } catch (InterruptedException e) {
       e.printStackTrace();
   } catch(ExecutionException e){
       e.printStackTrace();
   }
   System.out.println("result = " + result);  
   executorService.shutdown();  
   ```

   随机返回一个完成的任务，从Task 1到3之中随机返回一个。

5. invokeAll()

   方法 `invokeAll()` 会调用存在于参数集合中的所有 Callable 对象，并且返回一个包含 Future 对象的集合，你可以通过这个返回的集合来管理每个 Callable 的执行结果。
   需要注意的是，任务有可能因为异常而导致运行结束，所以它可能并不是真的成功运行了。但是我们没有办法通过 Future 对象来了解到这个差异。

   ```java
   List<Future<String>> futures = executorService.invokeAll(callables);
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 1";  
       }  
   });  
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 2";  
       }  
   });  
   callables.add(new Callable<String>() {  
       public String call() throws Exception {  
           return "Task 3";  
       }  
   });
   for (Future<String> afuture : futures) {
       try {
           System.out.println("future.get = " + afuture.get());
       } catch (ExecutionException e) {
           e.printStackTrace();
       }
   }
   ```
## 关闭ExecutorService

一般情况下，ExecutorService 并不会自动关闭，即使所有任务都执行完毕，或者没有要处理的任务，也不会自动销毁 ExecutorService 。它会一直出于等待状态，等待我们给它分配新的工作。

这种机制，在某些情况下是非常有用的，比如，，如果应用程序需要处理不定期出现的任务，或者在编译时不知道这些任务的数量。

但另一方面，这也带来了副作用：即使应用程序可能已经到达它的终点，但并不会被停止，因为等待的 ExecutorService 将导致 JVM 继续运行。这样，我们就需要主动关闭 ExecutorService。

要正确的关闭 ExecutorService，可以调用实例的 `shutdown()` 或 `shutdownNow()` 方法。

1. `shutdown()` 方法：

   ```java
   executorService.shutdown();
   ```

   `shutdown()` 方法并不会立即销毁 ExecutorService 实例，而是首先让 ExecutorService 停止接受新任务，并在所有正在运行的线程完成当前工作后关闭。

2. `shutdownNow()` 方法：

   ```java
   List<Runnable> notExecutedTasks = executorService.shutDownNow();
   ```

   `shutdownNow()` 方法会尝试立即销毁 ExecutorService 实例，所以并不能保证所有正在运行的线程将同时停止。该方法会返回等待处理的任务列表，由开发人员自行决定如何处理这些任务。

因为提供了两个方法，因此关闭 ExecutorService 实例的最佳实战 （ 也是 Oracle 所推荐的 ）就是同时使用这两种方法并结合 `awaitTermination()` 方法。

使用这种方式，ExecutorService 首先停止执行新任务，等待指定的时间段完成所有任务。如果该时间到期，则立即停止执行。

```java
executorService.shutdown();
try {
    if (!executorService.awaitTermination(800, TimeUnit.MILLISECONDS)) {
        executorService.shutdownNow();
    } 
} catch (InterruptedException e) {
    executorService.shutdownNow();
}
```

# Future接口

Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。

Future类位于`java.util.concurrent`包下，它是一个接口：

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- `cancel`方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数`mayInterruptIfRunning`表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论`mayInterruptIfRunning`为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若`mayInterruptIfRunning`设置为true，则返回true，若`mayInterruptIfRunning`设置为false，则返回false；如果任务还没有执行，则无论`mayInterruptIfRunning`为true还是false，肯定返回true。
- `isCancelled`方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- `isDone`方法表示任务是否已经完成，若任务完成，则返回true；
- `get()`方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
- `get(long timeout, TimeUnit unit)`用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

也就是说Future提供了三种功能：

1. 判断任务是否完成；
2. 能够中断任务；
3. 能够获取任务执行结果。

因为Future只是一个接口，所以是无法直接用来创建对象使用的，因此就有了下面的`FutureTask`。

# FutureTask

我们先来看一下`FutureTask`的实现：

```java
public class FutureTask<V> implements RunnableFuture<V>
```

`FutureTask`类实现了`RunnableFuture`接口，我们看一下`RunnableFuture`接口的实现：

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

可以看出`RunnableFuture`继承了`Runnable`接口和`Future`接口，而`FutureTask`实现了`RunnableFuture`接口。所以它既可以作为`Runnable`被线程执行，又可以作为`Future`得到`Callable`的返回值。

`FutureTask`提供了2个构造器：

```java
public FutureTask(Callable<V> callable) {
}
public FutureTask(Runnable runnable, V result) {
}
```

事实上，FutureTask是Future接口的一个唯一实现类。

使用Callable+FutureTask:

```java
public class Test {
    public static void main(String[] args) {
        //第一种方式
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executor.submit(futureTask);
        executor.shutdown();
         
        //第二种方式，注意这种方式和第一种方式效果是类似的，只不过一个使用的是ExecutorService，一个使用的是Thread
        /*Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        Thread thread = new Thread(futureTask);
        thread.start();*/
         
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
         
        System.out.println("主线程在执行任务");
         
        try {
            System.out.println("task运行结果"+futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
         
        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
}
```

---



参考资料：    
[线程池学习总结]([https://snailclimb.gitee.io/javaguide/#/./docs/java/Multithread/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93](https://snailclimb.gitee.io/javaguide/#/./docs/java/Multithread/java线程池学习总结))

[ExecutorService总结](https://www.twle.cn/c/yufei/javatm/javatm-basic-executorservice.html)  

[Java并发编程：Callable、Future和FutureTask](https://www.cnblogs.com/dolphin0520/p/3949310.html) 版权所有人[Matrix海子](http://www.cnblogs.com/dolphin0520/)

[四种线程池用法分析](https://blog.csdn.net/u011974987/article/details/51027795)