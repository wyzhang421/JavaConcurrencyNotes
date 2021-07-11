# Java Concurrency Notes

[TOC]

## 背景

本笔记来自于狂神说JUC笔记。

B站link: https://www.bilibili.com/video/BV1B7411L7tE

## 什么是JUC

```
java.util.concurrent
java.util.concurrent.atomic
java.util.concurrent.locks
```

java.util 工具包，包，分类

`Runnable`  没有返回值，效率相比于 Callable 相对较低

## 进程和线程回顾

### 线程，进程，如果不能用一句话说出来的技术，不扎实

进程：一个程序，QQ.exe, Music.exe 程序的集合

一个进程往往可以额包含多个线程，至少包含一个线程

Java 默认有几个线程？ 2个。 main 和 GC

线程：开了一个进程 Typora，写字，自动保存（线程负责的）

对于Java而言： Thread，Runnable， Callable

**Java真的可以开启线程吗？** 开不了 

```java
  AtomicInteger x = new AtomicInteger();
  new Thread(()->{ for (int i = 0; i < 40; ++i) System.out.println(x.getAndIncrement());}, "B").start();
```


<details>
<summary>thread start0()</summary>

```java
    /**
     * Causes this thread to begin execution; the Java Virtual Machine
     * calls the <code>run</code> method of this thread.
     * <p>
     * The result is that two threads are running concurrently: the
     * current thread (which returns from the call to the
     * <code>start</code> method) and the other thread (which executes its
     * <code>run</code> method).
     * <p>
     * It is never legal to start a thread more than once.
     * In particular, a thread may not be restarted once it has completed
     * execution.
     *
     * @exception  IllegalThreadStateException  if the thread was already
     *               started.
     * @see        #run()
     * @see        #stop()
     */
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0();
```

</details>

### 并发，并行

并发编程：并发，并行

并发（多线程操作同一个资源）

- CPU一核，模拟出来多条线程，天下武功，唯快不破，快速交替 

并行（多个人一起行走）

- CPU多核，多个线程可以同时执行；线程池

```java
    private static void getAvailableProcessors() {
        //获取CPU的核数
        System.out.println("available processors: " + Runtime.getRuntime().availableProcessors());
    }
```

并发编程的本质：**充分利用CPU的资源**

所有的公司都很看重！

企业，挣钱 => 提高效率，裁员，找一个厉害的人顶替三个不怎么样的人

人员（减），技术成本（高）

### 线程有几个状态 - 6个

<details>
<summary>thread state</summary>

```java
    /**
     * A thread state.  A thread can be in one of the following states:
     * <ul>
     * <li>{@link #NEW}<br>
     *     A thread that has not yet started is in this state.
     *     </li>
     * <li>{@link #RUNNABLE}<br>
     *     A thread executing in the Java virtual machine is in this state.
     *     </li>
     * <li>{@link #BLOCKED}<br>
     *     A thread that is blocked waiting for a monitor lock
     *     is in this state.
     *     </li>
     * <li>{@link #WAITING}<br>
     *     A thread that is waiting indefinitely for another thread to
     *     perform a particular action is in this state.
     *     </li>
     * <li>{@link #TIMED_WAITING}<br>
     *     A thread that is waiting for another thread to perform an action
     *     for up to a specified waiting time is in this state.
     *     </li>
     * <li>{@link #TERMINATED}<br>
     *     A thread that has exited is in this state.
     *     </li>
     * </ul>
     *
     * <p>
     * A thread can be in only one state at a given point in time.
     * These states are virtual machine states which do not reflect
     * any operating system thread states.
     *
     * @since   1.5
     * @see #getState
     */
    public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```
</details>


### wait/sleep 区别

**1. 来自不同的类**

wait => Object

sleep => Thread

**2. 关于锁的释放**

wait 会释放锁，sleep睡觉了，抱着锁睡觉，不会释放锁

**3. 使用的范围是不同的**

wait 必须在同步代码块中

sleep 可以在任何地方睡

**4. 是否需要捕获异常**

wait 不需要捕获异常

sleep 必须要捕获异常


## Lock锁

### 传统 Synchronized

```java
/**
 * 真正的多线程开发，公司中的开发，降低耦合性
 * 线程就是一个单独的资源类，没有任何附属的操作!
 * 1、 属性、方法
 */
public class SaleTicketDemo01 {
    public static void main(String[] args) {

        // 并发:多线程操作同一个资源类, 把资源类丢入线程
        Ticket ticket = new Ticket();

        // @FunctionalInterface 函数式接口，jdk1.8 lambda表达式 (参数)->{ 代码 }
        new Thread(()->{ for (int i = 0; i < 5; ++i) ticket.sale();}, "threadA").start();
        new Thread(()->{ for (int i = 0; i < 2; ++i) ticket.sale();}, "threadB").start();
        new Thread(()->{ for (int i = 0; i < 5; ++i) ticket.sale();}, "threadC").start();
    }
}

// 资源类 OOP
class Ticket {
    // 属性、方法
    private int number = 10;

    // 卖票的方式
    // synchronized 本质: 队列，锁
    public synchronized void sale() {
        if (number > 0) {
            System.out.println(Thread.currentThread().getName() + ": sold number " + number-- + "th, left: " + number);
        }
    }
}
```

### Lock 接口

```java
Lock l = ...; 
l.lock();
try {
    // access the resource proteced by this lock 
} finally {
    l.unlock();
}
```

**可重入锁（常用）**

```java
ReentrantLock
ReentrantReadWriteLock.ReadLock // 读锁
ReentrantREadWriteLock.WriteLock // 写锁
```

``` java
/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync(); // 非公平锁
}
/**
 * Creates an instance of {@code ReentrantLock} with the
 * given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantLock(boolean fair) { // 用构造参数判读是否是公平锁
    sync = fair ? new FairSync() : new NonfairSync();  // FairSync() 是公平锁
}
```

公平锁： 十分公平：可以先来后到

非公平锁： 十分不公平： 可以插队（默认）

```java
// Lock三部曲
// 1、 new ReentrantLock();
// 2、 lock.lock(); // 加锁
// 3、 finally=> lock.unlock(); // 解锁
```

```java
public class SaleTicketDemo02 {
    public static void main(String[] args) {
        // 并发:多线程操作同一个资源类, 把资源类丢入线程
        Ticket2 ticket2 = new Ticket2();

        // @FunctionalInterface 函数式接口，jdk1.8 lambda表达式 (参数)->{ 代码 }
        new Thread(()->{ for (int i = 0; i < 5; ++i) ticket2.sale();}, "threadA").start();
        new Thread(()->{ for (int i = 0; i < 2; ++i) ticket2.sale();}, "threadB").start();
        new Thread(()->{ for (int i = 0; i < 5; ++i) ticket2.sale();}, "threadC").start();
    }
}

// Lock三部曲
// 1、 new ReentrantLock();
// 2、 lock.lock(); // 加锁
// 3、 finally=> lock.unlock(); // 解锁
class Ticket2 {
    private int number = 10;
    Lock lock = new ReentrantLock();

    public void sale() {
        lock.lock(); // 加锁
        try {
            // 业务代码
            if (number > 0) {
                System.out.println(Thread.currentThread().getName() + ": sold number " + number-- + "th, left: " + number);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock(); // 解锁
        }
    }
}
```

### Synchronized 和 Lock 的区别

1. Synchronized 是内置的 Java 关键字， Lock 是一个 Java 类；
2. Synchronized 无法获取锁的状态，Lock 可以判断是否获取到了锁；
3. Synchronized 会自动释放锁，Lock 必须要手动释放锁。 如果不释放锁，那么会**死锁**；
4. Synchronized 线程1 （获得锁，阻塞），线程2（等待，傻傻的等）; Lock 锁不一定会等待下去；
5. Synchronized 可重入锁，不可以中断的，非公平； Lock，可重入锁，可以判断锁， 非公平（可以自自己设置）；
6. Synchronized 适合锁少量的代码同步问题，Lock 适合锁大量的同步代码


## 生产者和消费者

面试的：单例模式，排序算法，生产者和消费者，死锁

### 生产者和消费者问题的 Synchronized 版本

```java
public class PcDemo1 {
    public static void main(String[] args) {
        Product1 product = new Product1();
        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                try {
                    TimeUnit.SECONDS.sleep(2);
                    product.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "threadA").start();

        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                try {
                    product.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "threadB").start();

        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                try {
                    TimeUnit.SECONDS.sleep(2);
                    product.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "threadC").start();


        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                try {
                    product.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "threadD").start();

    }
}

class Product1 {
    private int number = 0;
    // 如果没有 synchronized 关键字， IllegalMonitorStateException
    public synchronized void increment() throws InterruptedException {
        // 这里如果用 if 的话，会有虚假唤醒的现象
        // https://www.cnblogs.com/jichi/p/12694260.html
        while (number != 0) {
            this.wait();
        }
        number++;
        System.out.println(Thread.currentThread().getName() + "=>" + number);
        this.notifyAll();
    }

    // 如果没有 synchronized 关键字， IllegalMonitorStateException
    public synchronized void decrement() throws InterruptedException {
        // 这里如果用 if 的话，会有虚假唤醒的现象
        // https://www.cnblogs.com/jichi/p/12694260.html
        while (number == 0) {
            this.wait();
        }
        number--;
        System.out.println(Thread.currentThread().getName() + "=>" + number);
        this.notifyAll();
    }
}

```

### JUC 版本的生产者和消费者的问题

Synchorinzed: wait, notify

Lock: await, signal

```java
public class PcDemo2 {
    public static void main(String[] args) {
        Product2 product2 = new Product2();

        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                product2.increment();
            }
        }, "A").start();

        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                product2.decrement();
            }
        }, "B").start();


        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                product2.increment();
            }
        }, "C").start();

        new Thread(()->{
            for (int i = 0; i < 10; ++i) {
                product2.decrement();
            }
        }, "D").start();
    }
}

// 判断等待，业务，通知
class Product2 { // 数字 资源类
    private int number = 0;

    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    // condition.await();  //等待
    // condition.signalALl(); // 唤醒全部

    public void increment() {
        lock.lock();
        try {
            // 判断等待
            while (number != 0) {
                condition.await();
            }
            // 业务
            number++;
            System.out.println(Thread.currentThread().getName() + "=>" + number);
            // 通知其他线程， 业务已经做完了
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void decrement() {
        lock.lock();
        try {
            // 判断等待
            while (number == 0) {
                condition.await();
            }
            // 业务
            number--;
            System.out.println(Thread.currentThread().getName() + "=>" + number);
            // 通知其他线程， 业务已经做完了
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

**任何一个新的技术，绝对不是仅仅只是覆盖了原来的技术，优势和补充!**

### Condition 精准的通知和唤醒线程



## 8锁现象

## 集合类不安全

## Callable

## CountDownLatch, CyclicBarrier, Semaphore

## 读写锁

## 阻塞队列

## 线程池

## 四大函数式接口

## Stream流式计算

## 分支合并

## 异步回调

## JMM

## volatile

## 深入单例模式

## 深入理解CAS

## 原子引用

