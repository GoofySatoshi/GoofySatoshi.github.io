---
title: JUC 并发工具类
date: 2025-10-25 02:00:00
categories: 并发编程
tags:
  - 并发工具类
  - ReentranLock
  - Semaphore
  - CountDownLatch
  - CyclicBarrier
  - Exchanger
  - Phaser
---
# 一、ReentranLock工具类

Java官方在早期jdk1.5版本就引入了ReentranLock并发工具类，这是一种可重入的独占锁（排它锁），它允许同一个线程多次获取同一个锁而不会被阻塞。相比synchronized，它提供了更灵活的控制能力，如可中断锁、可超时获取锁、条件变量等，用于解决高并发场景下需要灵活控制的业务场景。
下面是使用ReentranLock的演示代码
``` Java
import java.util.concurrent.locks.ReentrantLock;
/**
 * ReentrantLock使用演示：多线程操作共享资源的线程安全控制
 */
public class ReentrantLockDemo {
    // 创建可重入锁实例（默认非公平锁，传入true可创建公平锁）
    private static final ReentrantLock lock = new ReentrantLock();
    // 共享资源：计数器
    private static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        // 创建5个线程，每个线程对计数器累加1000次
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    increment(); // 调用加锁的累加方法
                }
            });
            threads[i].start();
        }
        // 等待所有线程执行完毕
        for (Thread thread : threads) {
            thread.join();
        }
        // 输出最终结果（预期5*1000=5000）
        System.out.println("最终计数：" + count);
    }

    /**
     * 加锁的累加方法（演示基础锁用法）
     */
    private static void increment() {
        lock.lock();// 1. 获取锁（必须在try外获取，避免获取锁前异常导致unlock错误）
        try {
            count++;// 2. 临界区：操作共享资源
            reentrantMethod();// 演示可重入特性：同一线程可再次获取锁（需对应释放）
        } finally {
            // 3. 释放锁（必须在finally中，确保锁一定被释放）
            lock.unlock();
        }
    }

    /**
     * 演示可重入特性：同一线程可再次获取已持有的锁
     */
    private static void reentrantMethod() {
        lock.lock(); // 再次获取锁（重入）
        try {
            // 可重入操作（例如日志记录等）
            System.out.println("重入方法执行");
        } finally {
            lock.unlock(); // 对应释放重入的锁
        }
    }
}
```
## 公平锁和非公平锁
ReentranLock支持公平锁和非公平锁两种模式，使用方式非常简单：
- 公平锁：线程在获取锁的时候，按照等待的先后顺序获取锁
- 非公平锁：线程在获取锁的时候，不按照等待的先后顺序获取锁，而是随机获取锁。ReentranLock默认是非公平锁
```Java
ReentranLock lock = new ReentranLock();//参数默认false,非公平锁
ReentranLock lock = new ReentranLock(true);//公平锁
```
## 可重入锁
可重入锁又名递归锁，是指同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象需要是同一个对象），不会因为之前已经获取过还没释放而阻塞。Java中ReentranLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。在实际开发中，可重入锁常常应用于递归操作、调用同一个类中的其他方法、锁嵌套等场景中。
```Java
class Counter{
    private final ReentranLock lock = new ReentranLock();//创建 ReentranLock

    public void recursiveCall(int num){
        lock.lock();
        try{
            if(num == 0){
                return ;
            }
            System.out.println("执行递归,num = " + num);
            recursiveCall(num - 1);
        } finally{
            lock.unlock();
        }
    }

    public static void main(String[] args){
        Counter test = new Counter();
        test.recursiveCall(10);
    } 
}
```
## 结合Condition实现生产者消费者
java.util.concurrent类库中提供Condition类来实现线程之间的协调/调用Condition.await()方法使线程等待，其他线程调用Condition.singal()或Condition.singnalAll()方法唤醒等待的线程。
注意：调用Condition的await()和signal()方法,都必须在lock保护之内。
```Java
import java.util.Random;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    // 独立的队列类
    static class Queue {
        private Object[] items;
        private int size = 0;
        private int putIndex = 0;
        private int takeIndex = 0;
        private ReentrantLock lock;
        private Condition notEmpty;  // 私有条件变量
        private Condition notFull;

        public Queue(int capacity) {
            this.items = new Object[capacity];
            lock = new ReentrantLock();
            notEmpty = lock.newCondition();
            notFull = lock.newCondition();
        }

        public void put(Object value) throws InterruptedException {
            lock.lock();
            try {
                while (size == items.length) {
                    notFull.await();  // 队列满则等待
                }
                items[putIndex] = value;
                if (++putIndex == items.length) {
                    putIndex = 0;
                }
                size++;
                notEmpty.signal();  // 唤醒消费者
                System.out.println("producer 生产:" + value); 
            } finally {
                lock.unlock();
            }
        }

        public Object take() throws InterruptedException {
            lock.lock();
            try {
                while (size == 0) {
                    notEmpty.await();  // 队列空则等待
                }
                Object value = items[takeIndex];
                items[takeIndex] = null;
                if (++takeIndex == items.length) {
                    takeIndex = 0;
                }
                size--;
                notFull.signal();  // 唤醒生产者
                return value;
            } finally {
                lock.unlock();
            }
        }
    }

    // 生产者线程
    static class Producer implements Runnable {
        private Queue queue;

        public Producer(Queue queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    Thread.sleep(1000);
                    queue.put(new Random().nextInt(1000));  // 生产随机数
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    // 消费者线程
    static class Consumer implements Runnable {
        private Queue queue;

        public Consumer(Queue queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    Thread.sleep(1000);
                    System.out.println("consumer 消费：" + queue.take());
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        Queue queue = new Queue(5);  // 创建队列
        new Thread(new Producer(queue)).start();  // 启动生产者
        new Thread(new Consumer(queue)).start();  // 启动消费者
    }
}
```

# 二、Semaphore 信号量
用于控制同时访问某个资源的线程数量

应用场景：
- 限流：可以用于限制对共享资源的并发访问数量，以控制系统的流量
- 资源池：可以用于实现资源池，以维护一组有限的共享资源
```Java
public class SemaphoreDemo{
    // 声明许可数量
    private static Semaphore semaphore = new Semaphore(2);
    private static Executor executor = new Executor(10);
    public static void main(String[] args) {
      for(int i=0;i<10;i++){
        executor.execute(()->getProductInfo());
      }
    }
    public static String getProductInfo(){
        try {
            // 申请许可
            semaphore.acquire();
            log.info("请求服务");
            Thread.sleep(2000);
        } catch(InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            // 释放许可
            semaphore.release;
        }
        return "返回商品详情信息";
    }
    // 限流算法
    public static String getProductInfo2(){
        // 尝试申请许可
        if(!semaphore.tryAcquire()){
            log.error("请求被流控了");
            return "请求被流控了";
        }
        try {
            log.info("请求服务");
            Thread.sleep(2000);
        } catch(InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            // 释放许可
            semaphore.release;
        }
        return "返回商品详情信息";
    }
}
```


## 三、CountDownLatch 闭锁
同步协助类，允许一个或多个线程等待，直到其他线程完成操作集
核心方法说明
- CountDownLatch(int count)：构造方法，初始化计数器计数器值。
- countDown()：计数器减 1（线程执行完后调用）。
- await()：当前线程阻塞，直到计数器变为 0。
- await(long timeout, TimeUnit unit)：带超时时间的等待，超时后即使计数器未到 0 也会继续执行。
- CountDownLatch 的计数器是一次性的，一旦计数器变为 0，再次调用 countDown() 也不会有任何效果。
应用场景：
- 百米赛跑，学生考试统一交卷，商品详情数据汇总。
- 并行任务同步：可以用于协调多个并行任务的完成情况,确保所有任务都完成后再继续执行下一步操作。
- 多任务汇总：可以用于统计多个线程的完成情况，以确定所有线程都已完成工作。
- 资源初始化：可以用于等待资源的初始化完成，以便在资源初始化后开始使用。
```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo2 {
    public static void main(String[] args) throws InterruptedException {
        int threadCount = 3;
        // 计数器初始化为1（主线程发出1个"开始"信号）
        CountDownLatch startSignal = new CountDownLatch(1);

        for (int i = 0; i < threadCount; i++) {
            final int runnerId = i + 1;
            new Thread(() -> {
                try {
                    System.out.println("选手 " + runnerId + " 准备就绪，等待发令...");
                    // 等待主线程的"开始"信号（计数器变为0）
                    startSignal.await();
                    // 收到信号后执行
                    System.out.println("选手 " + runnerId + " 开始跑步！");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        // 主线程程准备3秒后发出"开始"信号
        Thread.sleep(3000);
        System.out.println("主线程：各就各位位，预备——跑！");
        // 计数器减1（变为0），唤醒所有等待的子线程
        startSignal.countDown();
    }
}
```
## 四、CyclicBarrier 回环栅栏/循环屏障
实现让一组线程等待至某个状态(屏障点)之后再全部同时执行，而且可以被重复使用
核心方法说明：
- CyclicBarrier(int parties)：构造方法，指定需要等待的线程数量（parties）。
- CyclicBarrier(int parties, Runnable barrierAction)：指定等待线程数 + 屏障动作（所有线程到达后执行）。
- await()：当前线程到达屏障并等待，直到所有线程到达或被中断。
- await(long timeout, TimeUnit unit)：带超时的等待，超时后抛出 TimeoutException。
- reset()：重置屏障计数器，让其可以重新使用（未到达的线程会收到 BrokenBarrierException）。
- getNumberWaiting()：获取当前已到达屏障的线程数。
- isBroken()：判断屏障是否被打破（如线程中断、超时等）。
应用场景：
- 批量数据处理，人满发车
- 多线程任务:可以用于将复杂的任务分配给多个线程执行，并在所有线程完成工作后触发后续操作
- 数据处理：可以用于协调多个线程间的数据处理，在所有线程处理完数据后触发后续操作

```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo2 {
    public static void main(String[] args) {
        int runnerCount = 2;
        // 初始化屏障，指定2个线程等待，以及屏障动作
        CyclicBarrier barrier = new CyclicBarrier(runnerCount, () -> {
            System.out.println("===== 本轮轮比赛开始！ =====");
        });

        // 启动2个运动员线程，循环参与3轮比赛
        for (int i = 0; i < runnerCount; i++) {
            final int runnerId = i + 1;
            new Thread(() -> {
                try {
                    for (int round = 1; round <= 3; round++) { // 3轮比赛
                        System.out.println("第" + round + "轮：运动员" + runnerId + "准备中...");
                        Thread.sleep((long) (Math.random() * 1000));
                        System.out.println("第" + round + "轮：运动员" + runnerId + "已就位");

                        // 等待其他运动员
                        barrier.await();

                        // 所有就位后，执行本轮比赛
                        System.out.println("第" + round + "轮：运动员" + runnerId + "冲刺！");
                        Thread.sleep(500); // 模拟比赛过程
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```
## 五、Exchanger 数据交换器
用于线程间协作的工具类，用于两个线程间交换数据
核心方法说明:
- Exchanger()：构造方法，创建一个用于交换数据的同步器。
- exchange(V x)：当前线程携带数据 x 到达交换点，阻塞等待另一个线程，交换数据后返回对方的数据。
- exchange(V x, long timeout, TimeUnit unit)：带超时时间的交换，超时未完成则抛出 TimeoutException。
应用场景：
- 交易场景（一手交钱，一手交货），对账场景
- 数据交换：在多线程环境中，两个线程可以通过Exchanger进行数据交换
- 数据采集：在数据采集系统中，可以使用Exchanger在采集线程和处理线程间进行数据交换。

```java
import java.util.concurrent.Exchanger;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class ExchangerDemo2 {
    public static void main(String[] args) {
        Exchanger<Integer> exchanger = new Exchanger<>();

        // 线程A：立即到达交换点，等待3秒
        new Thread(() -> {
            try {
                int data = 100;
                System.out.println("线程A准备交换：" + data + "（最多等3秒）");
                // 超时等待：3秒后若线程B未到达，则抛出TimeoutException
                Integer received = exchanger.exchange(data, 3, TimeUnit.SECONDS);
                System.out.println("线程A收到交换的数据：" + received);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                System.out.println("线程A：等待超时，未完成交换");
            }
        }).start();

        // 线程B：延迟5秒到达（超过线程A的等待时间）
        new Thread(() -> {
            try {
                Thread.sleep(5000); // 延迟5秒
                int data = 200;
                System.out.println("线程B准备交换：" + data);
                // 此时线程A已超时退出，线程B会一直阻塞吗？
                // 不会：若交换失败（如对方超时），此处会抛出InterruptedException
                Integer received = exchanger.exchange(data);
                System.out.println("线程B收到交换的数据：" + received);
            } catch (InterruptedException e) {
                System.out.println("线程B：交换失败（对方已超时）");
            }
        }).start();
    }
}
```
# 六、Phaser 阶段协同器
CyclicBarrier 和 CountDownLatch的进化版，管理多个阶段的执行，可以让程序员灵活地控制线程的执行顺序和阶段性的执行
核心方法：
- Phaser(int parties)	构造方法，指定初始参与线程数（parties）
- register()	注册 1 个新线程参与（返回当前阶段号）
- bulkRegister(int parties)	批量注册多个线程（返回当前阶段号）
- arriveAndAwaitAdvance()	当前线程完成当前阶段，等待其他线程，所有线程到达后进入下一阶段
- arriveAndDeregister()	线程完成当前阶段并注销（不再参与后续阶段），返回当前阶段号
- arrive()	线程完成当前阶段但不等待（用于无需阻塞的场景）
- getPhase()	获取当前阶段号（从 0 开始）
- getRegisteredParties()	获取当前注册的线程数
- isTerminated()	判断是否已终止（onAdvance() 返回 true 后为 true）
- onAdvance(int phase, int parties)	回调方法，阶段切换时执行，返回 true 表示终止，false 继续下一阶段
应用场景：
- 多线程任务分配
- 多级任务流程
- 模拟并行计算
- 阶段性任务

```java
import java.util.concurrent.Phaser;

public class PhaserDemo1 {
    public static void main(String[] args) {
        int threadCount = 3;
        // 初始化Phaser：参数为初始参与线程数（可动态增减）
        Phaser phaser = new Phaser(threadCount) {
            // 重写onAdvance方法，定义阶段切换逻辑（返回true表示终止，false继续）
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("\n===== 阶段" + phase + "完成，共" + registeredParties + "个线程参与 =====");
                // 当所有阶段完成（这里设置3个阶段）或无参与线程时终止
                return phase >= 2 || registeredParties == 0;
            }
        };

        for (int i = 0; i < threadCount; i++) {
            final int threadId = i + 1;
            new Thread(() -> {
                // 模拟3个阶段的任务
                for (int phase = 0; !phaser.isTerminated(); phase++) {
                    System.out.println("线程" + threadId + " 正在执行阶段" + phase);
                    try {
                        // 模拟任务执行时间
                        Thread.sleep((long) (Math.random() * 1000));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    // 通知当前阶段完成，等待其他线程
                    phaser.arriveAndAwaitAdvance();
                }
                System.out.println("线程" + threadId + " 所有阶段完成");
            }).start();
        }
    }
}
```
