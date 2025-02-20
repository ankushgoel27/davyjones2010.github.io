---
title: Java线程BLOCKED与WAITING状态深入研究
date: 2023-02-25 17:01:22
tags: [java, javase, thread, sync, concurrent, know-why]
---

# 背景
上次机房断网的jstack分析之后, 发现其实个人并没有深入理解Java中线程的如下两个状态的区别: 
- "BLOCKED"
- "WAITING"
或者, 都是线程被阻塞无法运行(让出了CPU的时间片)的状态:
- 问题1: 这两个状态具体有啥区别? 
- 问题2: JVM为什么要进行上边两个状态的区分? 为什么不只用一个状态标识?

先不急着回答这个问题, 我们从一个例子出发:  

# 生产者&消费者例子
先来一道面试里时常问到的题目:
`两个线程, 分别扮演消费者&生产者的角色, 假设队列为1, 无限循环. 如何写?`

## 代码样例
- 完整代码参见: [ProducerConsumerTest.java](https://github.com/DavyJones2010/test-core/blob/master/src/test/java/edu/xmu/test/concurrent/ProducerConsumerTest.java)

```java
// consumer
while (true) {
    synchronized (lock) {
        while (isEmpty) {
            System.out.println("consumer is waiting");
            lock.wait();
        }
        System.out.println("start consuming");
        Thread.sleep((long) (Math.random() * 10000L));
        System.out.println("finished consuming");
        isEmpty = true;
        lock.notify();
    }
}

// producer
while (true) {
    synchronized (lock) {
        while (!isEmpty) {
            System.out.println("producer is waiting");
            lock.wait();
        }
        System.out.println("start producing");
        Thread.sleep((long) (Math.random() * 10000L));
        isEmpty = false;
        System.out.println("finished producing");

        lock.notify();
    }
}
```

## 变体写法-1
如果在调用`lock.notify()`之后再生产或者再消费, 会怎么样?
即代码变体如下: 

```java
// consumer
while (true) {
    synchronized (lock) {
        while (isEmpty) {
            System.out.println("consumer is waiting");
            lock.wait();
        }
        lock.notify();
        
        System.out.println("start consuming");
        Thread.sleep((long) (Math.random() * 10000L));
        System.out.println("finished consuming");
        isEmpty = true;
    }
}

// producer
while (true) {
    synchronized (lock) {
        while (!isEmpty) {
            System.out.println("producer is waiting");
            lock.wait();
        }
        lock.notify();

        System.out.println("start producing");
        Thread.sleep((long) (Math.random() * 10000L));
        isEmpty = false;
        System.out.println("finished producing");
    }
}

```

> 可以自己尝试下, 代码结果仍然是正常的, 即与正常写法完全没有区别. 

这是怎么回事儿? 
- producer调用`lock.notify()`的时候是直接把consumer唤醒开始执行了么? 但producer生产的代码在`lock.notify()`之后, 那么`isEmpty=false`是否会执行到?
- 如果`isEmpty=false`执行不到, 而是在consumer侧执行, 那么consumer也还是无法开始消费(因为无法走出`while(isEmpty)`这个判断循环), 结果应该是consumer仍然卡在`wait()`上. 
- 如果`isEmpty=true`执行到, 那么是在什么时候唤醒了consumer呢? 
好复杂, **代码的执行链路到底是怎样的??** 

## wait/notify详解
要搞清楚代码链路是怎样的, 首先我们需要清楚调用wait/notify时到底发生了什么?
> 由于wait/notify都是native的代码, 阅读不易. 因此下文主要内容/概念都是参照[Java SE Specification](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html), 进行个人理解与研究.

先给出三个重要的概念: 
- object monitor: 每个java对象都有一个object monitor
- blocked set: 每个java对象都有一个blocked set
- wait set: 每个java对象都有一个wait set

再来说明下三个概念实际是怎么使用的:
1. 如下简单代码, 实际线程执行时发生了如下事情: 
```java
Object o = new Object();
synchronized(o) {
    // do smt
}
```

- 线程执行`synchronized(o)`会导致: 
  - 线程尝试去抢占`object monitor`; 
  - 如果抢占到, 则该线程拥有了该`object monitor`, 进行临界区代码执行. 
  - 如果抢占不到, 则该线程被放入该对象的`blocked set`中. 具体 
    1. 什么时候唤醒: 可以简化认为为JVM会在底层时刻轮询`object monitor`占用情况, 一旦`object monitor`被释放, 立刻从`blocked set`中找个线程开始执行.
    2. 如果多个线程都在`block set`中, 该唤醒哪个, 由JVM来决定(TODO: 这个具体待探究, 不影响本文). 

- 线程退出`synchronized(o)`临界区会导致: 
  - 当前线程释放掉`object monitor`
  - JVM轮询到`object monitor`处于空闲, 立刻从`blocked set`中取出一个线程, 让该线程开始临界区代码执行.


2. 再加上简单的wait/notify: 
如下, 一个最简单的producer/consumer程序: 
```java
// consumer
Object o = new Object();
synchronized(o) {
    o.wait();
    // consume
}
// produce
synchronized(o) {
    // produce
    o.notify();
}
```

> 先假设consumer先执行

- consumer执行链路: 
  1. `synchronized(o)`: 获取到`object monitor`, 开始临界区代码执行.
  2. `o.wait()`: 虽然可以拆解为如下几步, 但wait本身是原子操作 
     1. 把consumer线程放入到该对象的 `wait set` 中
     2. 释放掉`object monitor`
     3. JVM将producer从`block set`中取出, (触发JVM/OS的轮询, 引发producer获取到`object monitor`从而进入临界区)
- producer执行链路:
  1. `synchronized(o)`: 获取不到`object monitor`, 被放入`block set`中 
  2. consumer退出临界区, producer被自动从`block set`中取出, 获取到`object monitor`从而进入临界区 (与consumer执行链路的2.3步骤重叠)
  3. `o.notify()`: 
    - producer并不会因为`notify()`而释放掉`object monitor`: <mark>**`notify`并不会导致当前线程释放掉`object monitor`!**</mark>, 而是继续往下执行代码.
    - JVM将consumer从`wait set`中取出, 尝试获得`object monitor`
    - 由于producer此时并没有释放掉`object monitor`, 因此JVM就把consumer放入到了`block set`中(即从`wait set`移到了`block set`中)
  4. 退出临界区: 
    - producer释放掉`object monitor`
    - JVM将consumer从`block set`中取出
    - 触发JVM内部的轮询, 引发consumer获取到`object monitor`, 从而继续`o.wait()`之后的代码片段执行
- consumer继续执行, 注意:
  1. 此时consumer是<mark>继续从`object.wait()`之后的代码开始执行. (即之前中断的地方继续).</mark> 
  2. 而<mark>不是重新通过`synchronized(o)`抢`object monitor`, 然后从头开始执行临界区代码.</mark> 因为内部JVM已经把`object monitor`给了consumer了.
  3. 退出临界区:
     - consumer释放掉`object monitor`
     - JVM尝试从`block set`中取出线程, 由于`block set`为空, nothing happens

- 终态: **producer执行完成, consumer也执行完成.**

> 再假设producer先执行

- producer执行链路:
  1. `synchronized(o)`: producer获取到`object monitor`, 开始临界区代码执行.

- consumer执行链路:
  1. `synchronized(o)`: consumer获取不到`object monitor`, 被放入`block set`中

- producer执行链路:
  1. `o.notify()`: 
     1. producer不会因为`notify()`而释放掉`object monitor`
     2. JVM从`wait set`中寻找一个线程, 移出`wait set`, 并放入到`blocked set`中. 由于此时`wait set`为空(consumer在`block set`中). 因此nothing happens 
  2. 退出临界区:
     1. producer释放掉`object monitor`
     2. JVM将consumer从`block set`中取出, 由于`object monitor`已经被producer释放, 因此consumer直接获取到`object monitor`, 开始执行临界区代码  (consumer状态 `block set` -> `RUNNABLE`)

- consumer执行链路:
  1. `o.wait()`: 
     1. 把consumer线程放入到该对象的 `wait set` 中
     2. 释放掉`object monitor`
     3. JVM尝试从`block set`中取出一个线程, 由于此时`block set`为空, 因此nothing happens

- 终态: **producer执行完成, consumer一直卡在`WAITING`状态.**

## wait/notify/synchronized总结

### 调用 synchronized(object) 时会发生: 
1. 当前线程尝试抢占 `object monitor`
2. 如果抢占到, 则进入临界区.
3. 如果抢占不到, 把当前线程放入到`blocked set`中. JVM会监控`object monitor`, 当`object monitor`归还时, 从`blocked set`中挑选一个线程继续代码执行(可能是进入临界区, 也可能是继续之前中断的代码)

### 出 synchronized(object) 时会发生:
1. 释放掉`object monitor`;
2. JVM会监控`object monitor`, 当它归还时, 从`blocked set`中挑选一个线程继续代码执行(可能是进入临界区, 也可能是继续之前中断的代码)

### 调用 object.wait() 时会发生:
1. 把当前线程放入到 `wait set` 中
2. 释放掉`object monitor`
3. JVM会监控`object monitor`, 当它归还时, 从`blocked set`中挑选一个线程继续代码执行(可能是进入临界区, 也可能是继续之前中断的代码)

### 调用 object.notify() 时会发生:
1. 不会因为`notify()`而释放掉`object monitor`, 而是继续往下执行代码.
2. JVM从`wait set`中寻找一个线程, 移出`wait set`, 并放入到`blocked set`中.(JVM会持续监控`object monitor`状态)


### 线程在不同位置的不同状态
因此可以根据线程所处的位置不同, 来区分不同状态: 

- 在`wait set`里: `WAITING`状态
- 在`blocked set`里: `BLOCKED`状态

因此也就从根本上解释了本文开头的第一个问题 `问题1: 这两个状态具体有啥区别?`

### 总结线程可能的状态变化

- `RUNNABLE` -> `block set`: 卡在synchronized
- `RUNNABLE` -> `wait set`: 卡在synchronized里wait
- `wait set`  -> `block set`: wait之后被其他线程调用的notify唤醒
- `block set` -> `wait set`: 不存在该链路, 可能当前线程在block set, 但被赋予object monitor之后, 肯定进入了RUNNABLE状态. 可能RUNNABLE之后主动调用了wait, 但也不是直接从`blocked set`到`wait set`
- `block set` -> `RUNNABLE`: 其他线程wait之后, 自动释放掉object monitor, 当前线程可以继续执行

```mermaid
stateDiagram
    [*] --> CREATED: new Thread
    CREATED --> RUNNABLE: Thread.start()
    RUNNABLE --> BLOCKED: synchronized()
    RUNNABLE --> WAITING: wait()
    BLOCKED --> RUNNABLE: 其他线程wait()
    WAITING --> BLOCKED: 其他线程调用notify()
    RUNNABLE --> DESTROYED: run执行结束, 或者抛出异常退出
    DESTROYED --> [*]: 结束

```

## wait/notify/synchronized实战
此时就很容易根据三个概念的流转, 来分析下上文的 `变体写法-1` 的执行流程, 也就明白为啥也可以work了.
本文就不再赘述了.

## 变体写法-2
如下例子中, 基于`变体写法-1`把`synchronized(lock)`放在`while(true)`外层, 会正常执行么? 可以先试着自己分析下(详细分析如下). 

```java
// consumer
synchronized (lock) { // 1. 获取object monitor
    while (true) {
        while (isEmpty) {
            System.out.println("consumer is waiting");
            lock.wait();  // 3. 把自己放到wait set里, 释放object monitor; 5. JVM把consumer从wait set里移出, 移入到block set里; 8. JVM把consumer从block set里移出, consumer获取object monitor
        }
        lock.notify(); // 9. 通知JVM把producer从wait set里移出, 移入到block set里; consumer继续往下执行(仍然保有object monitor)
        
        System.out.println("start consuming");
        Thread.sleep((long) (Math.random() * 10000L));
        System.out.println("finished consuming");
        isEmpty = true; // 10. consumer消费完成, 继续执行到第3步, 依次循环.
    }
}

// producer
synchronized (lock) { // 2. 把自己放到block set里, 等待获取object monitor; 4. 从block set移出, 获取 object monitor
    while (true) {
        while (!isEmpty) {
            System.out.println("producer is waiting"); 
            lock.wait(); // 7. 把自己放到wait set里, 释放object monitor; 9. JVM把producer从wait set里移出, 移入到block set里; 4.2 producer从block set移出, 获取 object monitor
        }
        lock.notify(); // 5. 通知JVM把consumer从wait set里移出, 移入到block set里; producer继续往下执行(仍然保有object monitor)

        System.out.println("start producing");
        Thread.sleep((long) (Math.random() * 10000L));
        isEmpty = false;  // 6. producer开始生产
        System.out.println("finished producing");
    }
}

```

> 答案揭晓: 没啥区别, 正常执行.

# 实际生产应用

// TODO: 

# 其他

## WAITING与TIMED_WAITING区别

## Object.wait() 与 Thread.sleep() 的区别
又是面试常问的一道题.  
通过上边分析可知

行为上区别: 
- Object.wait()之后:
  1. 把当前线程移到wait set里 
  2. 释放掉object monitor
  3. 线程暂停执行, 让出CPU时间片
- Thread.sleep()之后:
  1. 线程暂停执行, 让出CPU时间片; 不会有1, 2的操作.

结果上区别: 
- Object.wait()之后: 线程进入 WAITING (on object monitor) 或者 TIMED_WAITING (on object monitor) 状态 
- Thread.sleep()之后: 线程进入 TIMED_WAITING (sleeping) 状态

## 其他几种情况
除了本文, 其实还会有多种情况会导致线程进入`BLOCKED`, `WAITING`状态, 如下: 
![](https://www.baeldung.com/wp-content/uploads/2018/02/Life_cycle_of_a_Thread_in_Java.jpg)

但本质已经讲清楚了, 大家其实可以看下`Thread.join()`的源代码, 分析下当主线程调用`t.join()`与`t.join(30000L)`时, 主线程的状态应该是什么?

```java
@Test
public void name() throws InterruptedException {
    MyThread t = new MyThread(); // run()里边就是
    t.start();
//    t.join();
//    t.join(30000L);
}

static class MyThread extends Thread {
    @Override
    public void run() {
        try {
            Thread.sleep(100000L);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

> 答案: t.join()时, 主线程是WAITING (on object monitor); t.join(30000L)时, 主线程状态是TIMED_WAITING(on object monitor) 

原因: 源代码分析即可
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202302252202526.png)

# 总结
几点感想: 
1. 借此机会, 解释了自己内心以来的长久的疑惑, 感觉很通透. 
   1. 根本原因: JVM/OS封装了太多东西, 例如本文内容, 如果不知道有上边三种东西, 根本无法under the hood彻底解释清楚问题.
   2. 最好方式: 还是自己去翻源代码; 
   3. 其次: 就是看官方手册. 这次官方doc其实讲得也非常清晰.
   4. 最后: 工科一定要去实践, 自己写一些小demo打打jstack很多问题一下子就清晰了.
2. 针对Java中线程状态切换, 很多资料其实讲得并不好, 认识都很浅显, 甚至有极大的误导性. 大家引以为戒.
例如: [Thread States in Java](https://www.javatpoint.com/thread-states-in-java) 的状态机图, 存在如下几个问题: 
   1. 在实际生产中, 我们的jstack里, 永远不会看到`RUNNING`状态的线程, 都是处于`RUNNABLE`的. 参见 [JavaSE Spec](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html)
   2. 没有详细区分`BLOCKED`与`WAITING`这两种状态, 而这两个状态也是我们在jstack里常见的, 也是大家都会有疑惑的, 也是希望本文给大家阐述清晰的.

例如: [Difference Between BLOCKED, WAITING, And TIMED_WAITING? Explained Through Real-Life Examples](https://dzone.com/articles/difference-between-blocked-waiting-timed-waiting-e)
    1. 虽然是基于生活的情况进行类比, 但还是没有根本性第解释清楚这三种状态的根本区别. 看完仍是一头雾水.
    2. **一定要对于这种使用类比来解释技术问题的文章抱有高度警惕**. 例如把docker类比集装箱, 把HTTP协议类比俩人谈话等. 都是know what的, 但know why与know how的知识才是我们真正应该掌握的.


# Refs
- [Java Language Specification - Chapter 17. Threads and Locks](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html)
- [SoF上一个有意思的wait-notify问题](https://stackoverflow.com/questions/39927299/in-java-if-a-thread-calls-notify-before-wait-how-does-this-not-cause-the-se)




