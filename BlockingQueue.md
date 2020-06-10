### 什么是阻塞队列？
 **阻塞队列（BlockingQueue）** 是一个支持两个附加操作的队列。<br>
这两个附加的操作是：
> 1. 在队列为空时，获取元素的线程会等待队列变为非空。
> 2. 当队列满时，存储元素的线程会等待队列可用。

阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

### 阻塞队列提供了四种处理方法：

| 方法\处理方式 | 抛出异常 | 返回特殊值 | 一直阻塞 | 超时退出 |
| :-: | :-: | :-: | :-: | :-: | :-: |
| 插入方法 | add(e) | offer(e) | put(e) | offer(e,time,unit) |
| 移除方法 | remove()| poll() | take() | poll(time,unit) |
| 检查方法 | element() | peek() | 不可用 | 不可用 |

- **抛出异常：** 是指当阻塞队列满时候，再往队列里插入元素，会抛出 IllegalStateException("Queue full") 异常。当队列为空时，从队列里获取元素时会抛出 NoSuchElementException 异常 。
- **返回特殊值：** 插入方法会返回是否成功，成功则返回true。移除方法，则是从队列里拿出一个元素，如果没有则返回 null。
- **一直阻塞：** 当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到拿到数据，或者响应中断退出。当队列空时，消费者线程试图从队列里take元素，队列也会阻塞消费者线程，直到队列可用。
- **超时退出：** 当阻塞队列满时，队列会阻塞生产者线程一段时间，如果超过一定的时间，生产者线程就会退出。

***

### 阻塞队列的实现原理
JDK使用通知模式实现。
所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。通过查看 JDK 源码发现 ArrayBlockingQueue 使用了 Condition 来实现，代码如下：
```java
private final Condition notFull;
private final Condition notEmpty;

public ArrayBlockingQueue(int capacity, boolean fair) {
        // 省略其他代码 
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
}

public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
}

public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return extract();
  } finally {
            lock.unlock();
        }
}

private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex);
        ++count;
        notEmpty.signal();
}
```
> 当我们往队列里插入一个元素时，如果队列不可用，阻塞生产者主要通过 LockSupport.park(this); 来实现

```java 
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)

            reportInterruptAfterWait(interruptMode);
    }
}
```
> 继续进入源码，发现调用 setBlocker 先保存下将要阻塞的线程，然后调用 unsafe.park 阻塞当前线程。

```java
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        unsafe.park(false, 0L);
        setBlocker(t, null);
    }
```

unsafe.park 是个 native 方法，代码如下:
``` java 
 public native void park(boolean isAbsolute, long time);
``` 
park 这个方法会阻塞当前线程，只有以下四种情况中的一种发生时，该方法才会返回。
- 与 park 对应的 unpark 执行或已经执行时。注意：已经执行是指 unpark 先执行，然后再执行的 park。
- 线程被中断时。
- 如果参数中的 time 不是零，等待了指定的毫秒数时。
- 发生异常现象时。这些异常事先无法确定。


### Java 里的阻塞队列
- **ArrayBlockingQueue：** 一个由数组结构组成的有界阻塞队列。
  > 此队列按照先进先出（FIFO）的原则对元素进行排序。支持公平锁和非公平锁。【注：每一个线程在获取锁的时候可能都会排队等待，如果在等待时间上，先获取锁的线程的请求一定先被满足，那么这个锁就是公平的。反之，这个锁就是不公平的。公平的获取锁，也就是当前等待时间最长的线程先获取锁】
- **LinkedBlockingQueue：** 一个由链表结构组成的有界阻塞队列。
  > 一个由链表结构组成的有界队列，此队列的长度为Integer.MAX_VALUE。此队列按照先进先出的顺序进行排序。(可看成无界)
- **PriorityBlockingQueue：** 一个支持优先级排序的无界阻塞队列。
  > 默认自然序进行排序，也可以自定义实现compareTo()方法来指定元素排序规则，不能保证同优先级元素的顺序。
- **DelayQueue：** 一个实现PriorityBlockingQueue实现延迟获取的无界队列。
  > 在创建元素时，可以指定多久才能从队列中获取当前元素。只有延时期满后才能从队列中获取元素。
  
  > DelayQueue可以运用在以下应用场景
  > 1. 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
  > 2. 定时任务调度。使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，从比如TimerQueue就是使用DelayQueue实现的。
  
  > **如何实现Delay接口**<br>
  > 参考 ScheduledThreadPoolExecutor 里 ScheduledFutureTask 类。这个类实现了 Delayed 接口。首先：在对象创建的时候，使用 time 记录前对象什么时候可以使用。

  > 延时队列的实现，当消费者从队列里获取元素时，如果元素没有达到延时时间，就阻塞当前线程。
  
- **SynchronousQueue：** 一个不存储元素的阻塞队列。
  > 每一个put操作必须等待take操作，否则不能添加元素。支持公平锁和非公平锁。SynchronousQueue的一个使用场景是在线程池里。Executors.newCachedThreadPool()就使用了SynchronousQueue，这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，线程空闲了60秒后会被回收。
- **LinkedTransferQueue：** 一个由链表结构组成的无界阻塞队列。
  相比于其它队列，LinkedTransferQueue队列多了transfer和tryTransfer方法。<br>
  transfer 方法。如果当前有消费者正在等待接收元素（消费者使用 take() 方法或带时间限制的 poll() 方法时），transfer 方法可以把生产者传入的元素立刻 transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer 方法会将元素存放在队列的 tail 节点，并等到该元素被消费者消费了才返回。

  transfer 方法的关键代码如下：
  ```java
  Node pred = tryAppend(s, haveData);
  return awaitMatch(s, pred, e, (how == TIMED), nanos);
  ```
  
  > 第一行代码是试图把存放当前元素的 s 节点作为 tail 节点。第二行代码是让 CPU 自旋等待消费者消费元素。因为自旋会消耗 CPU，所以自旋一定的次数后使用 Thread.yield() 方法来暂停当前正在执行的线程，并执行其他线程。

  tryTransfer 方法。则是用来试探下生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回 false。和 transfer 方法的区别是 tryTransfer 方法无论消费者是否接收，方法立即返回。而 transfer 方法是必须等到消费者消费了才返回。

  对于带有时间限制的 tryTransfer(E e, long timeout, TimeUnit unit) 方法，则是试图把生产者传入的元素直接传给消费者，但是如果没有消费者消费该元素则等待指定的时间再返回，如果超时还没消费元素，则返回 false，如果在超时时间内消费了元素，则返回 true。

- **LinkedBlockingDeque：** 一个由链表结构组成的双向阻塞队列。
  > 队列头部和尾部都可以添加和移除元素，多线程并发时，可以将锁的竞争最多降到一半。相比其他的阻塞队列，LinkedBlockingDeque 多了 addFirst，addLast，offerFirst，offerLast，peekFirst，peekLast 等方法，以 First 单词结尾的方法，表示插入，获取（peek）或移除双端队列的第一个元素。以 Last 单词结尾的方法，表示插入，获取或移除双端队列的最后一个元素。另外插入方法 add 等同于 addLast，移除方法 remove 等效于 removeFirst。但是 take 方法却等同于 takeFirst，不知道是不是 Jdk 的 bug，使用时还是用带有 First 和 Last 后缀的方法更清楚。

  在初始化 LinkedBlockingDeque 时可以设置容量防止其过渡膨胀。另外双向阻塞队列可以运用在“工作窃取”模式中。
