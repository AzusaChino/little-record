# 并发类的面试

## 1.线程创建

>有哪些方法创建线程

创建有且只有一个new Thread()
运行线程有两种 -> Runnable.run() & Thread.start()

```java
public static void main(String[] args) {
    // main 线程 -> 子线程
    Thread t = new Thread(() -> {}, "subThread-1");
}

```

>如何创建进程

```java
public static void main(String[] args) {
    // 获取Java Runtime
    Runtime runtime = Runtime.getRuntime();
    Process process = runtime.exec("calc");
    process.exitValue();
}
```

>如何销毁一个线程

```java
public static void main(String[] args) {
    // main 线程 -> 子线程
    Thread t = new Thread(() -> {
        System.out.printf("thread [%s] is running\n", Thread.currentThread().getName());

    }, "subThread-1");
    // 启动线程
    t.start();
    System.out.printf("thread [%s] is alive: %s\n", t.getName(), t.isAlive());
    // 在java中, 执行线程java是没有办法销毁它的,
    // 但是当Thread.isAlive()返回false时, 实际底层的thread已经被销毁了
}
```

## 2.线程执行

>如何通过JAVA API启动线程

```java
thread.start()
```

> 如何让线程顺序执行

在线程start()之后立刻join()

```java
public static void main(String[] args) {
    Thread t1 = new Thread(Demo::action, "t1");
    Thread t2 = new Thread(Demo::action, "t2");
    Thread t3 = new Thread(Demo::action, "t3");
    // start() 仅是通知线程启动, 不保证执行顺序
    t1.start();
    t2.start();
    t3.start();
    // join() 控制线程必须执行完成, 不保证执行顺序
    t1.join();
    t2.join();
    t3.join();
    // start() + join() 可以保证执行顺序
    t1.start();
    t1.join();
}

private static void action() {
    System.out.printf("Thread[%s] is running...\n", Thread.currentThread().getName());
}
```

>其他的实现方式

```java

    Thread t1 = new Thread(Demo::action, "t1");
    Thread t2 = new Thread(Demo::action, "t2");
    Thread t3 = new Thread(Demo::action, "t3");
private static void threadLoop() {
    t1.start();
    while(t1.isAlive()) {
        // 自旋 self Spin
    }
    t2.start();
    while(t2.isAlive()) {}
    t3.start();
    while(t3.isAlive()) {}
}
private static void threadSleep() {
    t1.start();
    while(t1.isAlive()) {
        // sleep
        Thread.sleep(0);
    }
    t2.start();
    while(t2.isAlive()) {
        Thread.sleep(0);
    }
    t3.start();
    while(t3.isAlive()) {
        Thread.sleep(0);
    }
}
private static void threadWait() {
    threadStartAndWait(t1);
    threadStartAndWait(t2);
    threadStartAndWait(t3);
}

private static void threadStartAndWait(Thread t) {
    if (Thread.State.New.equals(t.getState())) {
        t.start();
    }
    while (t.isAlive()) {
        synchronized (t) {
            try {
                t.wait(); // 谁通知了Thread -> thread.notify
            } catch(InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }
}

```

## 3.线程中止

>如何停止一个线程? (没办法停止线程, 只能停止逻辑)

```java
public static void main(String[] args) {
    Action a = new Action();
    Thread t = new Thread(a, "t1");
    t.start();
    a.setStopped(true);
    t.join();

    Thread t2 = new Thread(() -> {
        if (!Thread.currentThread().isInterrupted()) {
            action();
        }
    }, "t2");
    t2.start();
    // 中断操作 (仅仅设置状态, 而并非中止线程)
    t2.interrupt();
    t2.join();
}

private static class Action implements Runnable {
    // 线程安全问题, 确保可见性 Happen-Before
    private boolean stopped = false;

    @override
    public void run(){
        if (!stopped) {
            action();
        }
    }

    public void setStopped(boolean stopped) {
        this.stopped = stopped;
    }
}

public static void action() {
    System.out.printf("Thread [%s] is running..", Thread.currentThread().getName());
}
```

>为什么JAVA放弃了Thread的stop方法

Inherently unsafe, 为了防止死锁, 状态不一致
Stopping a thread causes it to unlock all the monitors that it has locked.(The monitors are unlocked as the ThreadDeath exception propagates up the stack)

>Thread interrupt(), isInterrupted()以及interrupted()的区别和意义

- interrupt 设置线程状态
- isInterrupted 仅判断不清除
- interrupted() 判断&清除

## 4.线程异常

>当线程遇到异常时, 到底发生了什么?

```java
public static void main(String[] args){
    Thread t = new Thread(() -> {
        throw new RuntimeException("eee");
    }, "t1");
    t1.start();
    t1.join();

    // java里的THread是一个包装, 由GC做垃圾回收
    // JVM Thread 可能是一个OS Thread, JVM管理, 当线程执行完毕(正常或异常)
    System.out.print(t1.isAlive()); // false (t1的对象还在, 线程已经终止了)
}
```

>当线程遇到异常时, 如何捕获?

```java
public static void main(String[] args){
    Thread.setDefaultUncaughtExceptionHandler((t, throwable) -> {
        System.out.printf("Thread [%s] runs with err, %s", t.getName(), throwable.getMessage());
    })
    Thread t = new Thread(() -> {
        throw new RuntimeException("eee");
    }, "t1");
    t1.start();
    t1.join();

    System.out.print(t1.isAlive()); // false
}
```

>当线程遇到异常时, ThreadPoolExecutor如何捕获异常?

```java
public static void main(String[] args){
    ExecutorService service = Executors.newFixedThreadPool(2);
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        1,1,0,TimeUnit.SECONDS, new LinkedBlockingQueue()
    ) {
        // 通过覆盖afterExecute达到获取异常的信息
        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            System.out.printf("Thread [%s] runs with err, %s", Thread.currentThread().getName(), t.getMessage());
        }
    }
    service.execute(()-> {
        throw new RuntimeException("eee");
    });

    // 等待一秒钟, 确保提交的任务完成
    service.awaitTermination(1, TimeUnit.SECONDS);
    // 关闭线程池
    service.shutdown();
}
```

## 5.线程状态

>1.线程有哪些状态, 分别代表什么含义

NEW, RUNNABLE, BLOCKED, WAITTING, TIMED_WAITING, TERMINATED;

>2.如何获取当前JVM所有的线程状态

jps获取java线程id -> jstack + pid

```java
public static void main(String[] args) {
    // 包含了所有线程的状态
    ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
    long[] threadIds = mxBean.getAllThreadIds();

    Stream.of(threadIds).forEach(id -> System.out.println(mxBean.getThreadInfo(id)));
}
```

>3.如何获取线程的资源消费情况

```java
public static void main(String[] args) {
    ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
    long[] threadIds = mxBean.getAllThreadIds();
    Stream.of(threadIds).forEach(id -> {
        ThreadInfo threadInfo = mxBean.getThreadInfo(id);
        // 占用了多少bytes
        long bytes = mxBean.getThreadAllocatedBytes(id);
        System.out.printf("Thread[%d], using %d bytes", thread, bytes)
    }));
}
```

## 6.线程同步

>请说明synchronized关键字在修饰方法和代码块中的作用

基本上没什么区别  
看字节码(class)  
方法 -> ACC_SYNCHRONIZED  
代码块 -> monitorenter  

>请说明synchronized关键字与ReentrantLock之间的关系

- 都可以重进入

>请解释偏向锁对synchronized与ReentrantLock的价值

偏向锁对ReentrantLock没有影响

## 7.线程通讯

>为什么wait()和notify()以及notifyAll()方法属于Object, 并解释它们的作用

待查询

>为什么Object wait()和notify()以及notifyAll()方法必须在synchronized之中执行

wait() 获得锁的对象, 释放锁, 当前线程又被阻塞 ->类似于 LockSupport park() (需显示唤醒)
notify() 已经获得锁, 唤起一个被阻塞的线程 -> unpark()

>请通过Java代码模拟实现wait()和notify()以及notifyAll()的语义

待实现

## 8.线程退出

>当主线程退出时, 守护子线程会执行完毕吗

主线程退出时, 守护子线程不一定会执行 (和执行时间有关系,但非唯一评判标准)

```java
// main thread
public static void main(String[] args) {
    // sub thread
    Thread t1 = new Thread(()-> System.out.println("Hello, World"));
    t1.setDaemon(true);
    t1.start();
}
```

>请说明Shutdown hook线程的使用场景, 以及如何触发执行

```java
public static void main(String[] args) {
    Runtime runtime = Runtime.getRuntime();
    runtime.addShutdownHook(new Thread(() -> {
        System.out.printf("Thread [%s] is running...", Thread.currentThread().getName());
    }, "shutDown"));
}
```

>如何确保主线程退出前, 所有线程执行完毕

```java
public static void main(String[] args) {
    Thread t1 = new Thread(()-> System.out.println("Hello, World"), "t1");
    t1.start();

    // other N threads

    Thread mainThread = Thread.currentThread();
    ThreadGroup threadGroup = mainThread.getThreadGroup();
    int count = threadGroup.activeCount();
    Thread[] threads = new Thread[count];
    // copy all threads to `threads` array
    threadGroup.enumerate(threads, true);

    for (Thread t: threads) {
        if (Thread.State.New.equals(t.getState())) {
            t.start();
            t.join();
        }
    }
}
```

## 9.线程安全集合

>请在Java集合框架以及J.U.C框架中各举出List,Set以及Map的实现

- ArrayList, LinkedList,CopyOnWriteArrayList
- HashSet, TreeSet, CopyOnWriteArraySet,ConcurrentListSet
- HashMap, TreeMap, ConcurrentHashMap

>如何将普通List, Set以及Map转化为线程安全对象

```java
public static void main(String[] args) {
    // java 8
    List<Integer> list = Arrays.asList(1,2,3,4,5);
    // java 9+ of, return immutable object
    Set<Integer> set = Set.of(1,2,3,4,5);
    Map<Integer, String> map = Map.of(1,"a");

    // 1. call Collections.synchronized*
    // by adding synchronized (Wrapper design pattern)
    list = Collections.synchronizedList(List);

    // 2. java.util.concurrent.*
    list = CopyOnWriteArrayList<>(list);
    set = CopyOnWriteArraySet<>(set);
    map = ConcurrentHashMap<>(list);
}
```

>如何在Java9+实现以上问题

List.of() // return immutable object

## 10.线程安全list

> 请说明List, Vector以及CopyOnWriteArrayList的相同点和不同点

- 相同点
  - 实现了List
- 不同点
  - CopyOnWriteArrayList 读的时候不加锁
  - Vector 全部是同步的(synchronized)
  - Vector iterate fast-fail

> 请说明Collections.synchronizedList(list)与Vector的相同点和不同点

- 相同点
  - 通过synchronized实现
- 不同点
  - 返回类型
  - Collections.synchronizedList使用了Wrapper模式

> Arrays.asList(Object ...)方法是线程安全的吗? 如果不是, 如何实现替代方案

- return value -> mutable -> thread unsafe
- java < 5 Collections.synchronizedList(list)
- java 5+ CopyOnWriteArrayList
- java 9+ List.of(...)

## 11.线程安全Set

> 请至少举出三种线程安全的Set的实现

- Collections.synchronizedSet(set)
- CopyOnWriteArraySet
- ConcurrentListSet

> 在J.U.C框架中, 存在HashSet的线程安全实现吗? 如果不存在的话, 要如何实现

没有

```java
// refer to the HashSet -> HashMap
private static class ConcurrentHashSet<E> implements HashMap<E> {
    // ConcurrentHashSet -> ConcurrentHashMap
    private transient ConcurrentHashMap<E, Object> map = new ConcurrentHashMap<>();

    private Set<E> ketSet() {
        return map.ketSet();
    }

    //skip all override functions

}
```

> 当Set.iterator()方法返回iterator对象后, 能否在其迭代中, 给Set对象添加新的元素

不一定, 看具体实现, 如果是Concurrent

## 12.线程安全MAP

> 请说明Hashtable, Hash Map 以及ConcurrentHashMap的区别

- Hashtable
  - 不允许key, value为空
  - 数组加列表实现
- HashMap
  - key, value都可以为空
  - 数组加红黑树
- ConcurrentHashMap
  - 不允许key, value为空

> 请说明ConcurrentHashMap在不同的JDK中的实现

- 1.8 加入了红黑树实现

> 请说明ConcurrentHashMap和ConcurrentSkipListMap各自的优势和不足

- ConcurrentHashMap 数据加锁
- ConcurrentSkipListMap 数据不加锁
  - 优势: 高性能
  - 不足: 占空间

## 13.线程安全QUEUE

> 请说明BlockingQueue和Queue的区别

- BlockingQueue继承了Queue
- 存取 都有阻塞处理

> 请说明LinkedBlockingQueue与ArrayBlockingQueue的区别

- Linked <> Array
- 无边界(Integer.MAX) <> 有边界(bounded)

> 请说明LinkedTransferQueue与LinkedBlockingQueue的区别

(1.7+)LinkedTransferQueue比LinkedBlockingQueue性能好

## 14.例题部分

> 请评估以下程序的运行结果

```java
    @SneakyThrows
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new PriorityBlockingQueue<>(2);
        queue.put(9);
        queue.put(8);
        queue.put(7);

        System.out.println("queue.size() = " + queue.size()); // 3
        System.out.println("queue.take() = " + queue.take()); // 7
        System.out.println("queue = " + queue); // [8,9]
    }
```

## 15. 锁LOCK

> 请说明ReentrantLock 与ReentrantReadWriteLock的区别

- 都是可重入锁
- 都是互斥锁

> 请解释ReentrantLock为什么命名为重进入

```java
    private static ReentrantLock LOCK = new ReentrantLock();

    public static void main(String[] args) {
        synchronizedFunction(ReentrantLockTest::action1);
    }

    private static void action1() {
        synchronizedFunction(ReentrantLockTest::action2);
    }

    private static void action2() {
        System.out.println("Hello, World");
    }

    public static void synchronizedFunction(Runnable r) {
        LOCK.lock();
        try {
            r.run();
        } finally {
            LOCK.unlock();
        }
    }
```

> 请说明Lock.lock() 与 Lock.lockInterruptibly()的区别

如果当前线程被其他线程调用了interrupt()方法式, 这时会返回true  
再通过selfInterrupt()方法再将状态恢复(interrupted = true)

```java
public static void main(String[] args) {
    try {
        lock.lockInterruptibly();
        action();
    } catch(InterruptedException e) {
        // 显式恢复中断状态
        Thread.currentThread().interrupt();
        // 当前线程
    } finally{
        lock.unlock();
    }
}
```

## 16. 条件变量 Condition

> 请举例说明Condition的使用场景

- 生产者-消费者问题

> 请使用Condition实现`生产者-消费者问题`

- Condition

>请解释Condition.await() 和signal() 与 Object.wait() 和notify()的区别

- 都是阻塞和释放  

Thread是特殊的Object, 当线程isAlive() false的时候, thread.wait()操作会被自动释放  
Java Thread对象和实际JVM执行的OS Thread不是相同对象  
JVM Thread回调Java Thread.run()方法  
同时 Thread提供一些native方法获取JVM Thread状态  
当JVM Thread执行后, 自动就notify()了  

## 17.屏障BARRIERS

> 请说明CountDownLatch与CylicBarrier的区别

```java
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch latch = new CountDownLatch(4);
        ExecutorService service = ThreadUtil.newExecutor(5);
        for (int i = 0; i < 5; i++) {
            service.submit(() -> {
                action();
                latch.countDown(); // -1
            });
        }
        // waiting for finish
        // when count > 0, blocking
        latch.await();
        System.out.println("two subThreads complete");
        // close executor
        service.shutdown();
    }

    public static void action() {
        System.out.printf("Thread[%s] is running...\n", Thread.currentThread().getName());
    }
```

> 请说明Semaphore的使用场景

Semaphore是一个计数信号量，常用于限制可以访问某些资源（物理或逻辑的）线程数目。

> 请通过java1.4语法实现一个CountDownLatch

```java
 private static class CountDownLatch {
        private int count;

        private CountDownLatch(int count) {
            this.count = count;
        }

        public void await() throws InterruptedException {
            // count > 0 wait
            if (Thread.interrupted()) {
                throw new InterruptedException();
            }
            synchronized (this) {
                while (count > 0) {
                    wait(); // block current thread
                }
            }
        }

        public void countDown() {
            synchronized (this) {
                if (count < 1) {
                    return;
                }
                count--;
                if (count == 0) {
                    notifyAll();
                }
            }
        }
    }
```

## 17.线程池

> 请说明JUC中内建了几种ExecutorService实现

- ThreadPoolExecutor
- ScheduledThreadPoolExecutor
- ForkJoinPool

> 请分别解释ThreadPoolExecutor构造器参数再运行时的作用

- corePoolSize： 线程池维护线程的最少数量
- maximumPoolSize：线程池维护线程的最大数量
- keepAliveTime： 线程池维护线程所允许的空闲时间
- unit： 线程池维护线程所允许的空闲时间的单位
- workQueue： 线程池所使用的缓冲队列
- handler： 线程池对拒绝任务的处理策略

> 如何获取ThreadPoolExecutor正在运行的线程

// 实现ThreadFactory 另外保存一份thread

```java
    public static void main(String[] args) {

        ExecutorService service = ThreadUtil.newExecutor(20);
        Set<Thread> set = new HashSet<>();

        setThreadFactory(service, set);

        for (int i = 0; i < 9; i++) {
            service.submit(() -> {
            });
        }

        set.stream().filter(Thread::isAlive)
                .forEach(t -> {
                    System.out.printf("current thread [%s] is running...\n", t.getName());
                });

        // main thread group is always the parent

        Thread mainThread = Thread.currentThread();
        ThreadGroup threadGroup = mainThread.getThreadGroup();

        int count = threadGroup.activeCount();
        Thread[] threads = new Thread[count];
        threadGroup.enumerate(threads);

        Stream.of(threads)
                .filter(Thread::isAlive)
                .forEach(t -> {
                    System.out.printf("main's subThread [%s] is running...\n", t.getName());
                });

        service.shutdown();
    }

    private static void setThreadFactory(ExecutorService service, Set<Thread> threads) {
        if (service instanceof ThreadPoolExecutor) {
            ThreadPoolExecutor executor = (ThreadPoolExecutor) service;
            ThreadFactory oldFactory = executor.getThreadFactory();
            executor.setThreadFactory(new DelegateThreadFactory(oldFactory, threads));
        }
    }

    private static class DelegateThreadFactory implements ThreadFactory {

        private final ThreadFactory threadFactory;

        private final Set<Thread> threads;

        private DelegateThreadFactory(ThreadFactory threadFactory, Set<Thread> threads) {
            this.threadFactory = threadFactory;
            this.threads = threads;
        }

        @Override
        public Thread newThread(Runnable r) {
            Thread t = threadFactory.newThread(r);
            threads.add(t);
            return t;
        }
    }
```

## 18.FUTURE

> 如何获取Future对象

实现callable, submit(r)

> 请举例Future.get() 以及get(long, TimeUnit)方法的使用场景

- 主要用于超时等待

> 如何利用Future优雅地取消一个线程的执行

```java
public static void main(String[] args) {
        ExecutorService service = ThreadUtil.newExecutor(10);

        Future<?> future = service.submit(() -> {
            // finish under 3 seconds is ok, otherwise timeout
            sleep(3);
        });
        try {
            future.get(3, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        } catch (TimeoutException e) {
            // timeout ,close
            Thread.currentThread().interrupt();
            // attempt to cancel
            future.cancel(true);
        }

        service.shutdown();
    }

    @SneakyThrows
    private static void sleep(long seconds) {
        Thread.sleep(TimeUnit.SECONDS.toMillis(seconds));
        if (Thread.interrupted()) {
            return;
        }
        action();
    }

    public static void action() {
        System.out.printf("Thread[%s] is running...\n", Thread.currentThread().getName());
    }
```

## 19. Volatile

> 在Java中, volatile保证的是原子性还是可见性

可见性(一定)和原子性(不一定 `原生类型都是原子性`)

> 在Java中, volatile long和double是线程安全的吗

是线程安全的

> 在Java中, volatile的底层实现是基于什么机制

内存屏障

## 20.原子操作 Atomic

> 为什么AtomicBoolean内部变量用int实现, 而非boolean

虚拟机 x86 和 x64 -> 32bit 的int

> 在变量原子操作时, Atomic*CAS操作比synchronized关键字, 哪个更重

- 偏向锁 < CAS < 重锁(完全互斥)
- CAS操作也是相对重的操作, 他也是实现synchronized 瘦锁(thin lock)的关键
- 偏向锁就是避免CAS (Compare And Set/Swap)操作
- 当没有锁竞争时, CAS比synchronzed重
- 当竞争多时, 看情况

> Atomic*CAS的底层是如何实现的

- compare and exchange