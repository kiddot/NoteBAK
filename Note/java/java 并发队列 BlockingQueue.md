# java 并发队列 BlockingQueue

## BlockingQueue

​	首先，最基本的来说， BlockingQueue 是一个**先进先出**的队列（Queue），为什么说是阻塞（Blocking）的呢？是因为 BlockingQueue 支持当获取队列元素但是队列为空时，会阻塞等待队列中有元素再返回；也支持添加元素时，如果队列已满，那么等到队列可以放入新元素时再放入。

​	BlockingQueue 是一个接口，继承自 Queue，所以其实现类也可以作为 Queue 的实现来使用，而 Queue 又继承自 Collection 接口。

​	BlockingQueue 对插入操作、移除操作、获取元素操作提供了四种不同的方法用于不同的场景中使用：1、抛出异常；2、返回特殊值（null 或 true/false，取决于具体的操作）；3、阻塞等待此操作，直到这个操作成功；4、阻塞等待此操作，直到成功或者超时指定时间。总结如下：

| operation | Throws Exception | Special Value | Blocks | Times Out                   |
| --------- | ---------------- | ------------- | ------ | --------------------------- |
| Insert    | add(o)           | offer(o)      | put(o) | offer(o, timeout, timeunit) |
| Remove    | remove(o)        | poll()        | take() | poll(timeout, timeunit)     |
| Examine   | element()        | peek()        |        |                             |



> ​	BlockingQueue 不接受 null 值的插入，相应的方法在碰到 null 的插入时会抛出 NullPointerException 异常。null 值在这里通常用于作为特殊值返回（表格中的第三列），代表 poll 失败。所以，如果允许插入 null 值的话，那获取的时候，就不能很好地用 null 来判断到底是代表失败，还是获取的值就是 null 值。



## ArrayBlockingQueue

​	ArrayBlockingQueue 是 BlockingQueue 接口的有界队列实现类，底层采用数组来实现。它采用一个 ReentrantLock 和相应的两个 Condition 来实现：

```java
// 用于存放元素的数组
final Object[] items;
// 下一次读取操作的位置
int takeIndex;
// 下一次写入操作的位置
int putIndex;
// 队列中的元素数量
int count;
// 以下几个就是控制并发用的同步器
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;
```

对于 ArrayBlockingQueue，我们可以在构造的时候指定以下三个参数：

1. 队列容量，其限制了队列中最多允许的元素个数；
2. 指定独占锁是公平锁还是非公平锁。非公平锁的吞吐量比较高，公平锁可以保证每次都是等待最久的线程获取到锁；
3. 可以指定用一个集合来初始化，将此集合中的元素在构造方法期间就先添加到队列中。



## LinkedBlockingQueue

​	底层基于单向链表实现的阻塞队列，可以当做无界队列也可以当做有界队列来使用。用了两个锁，两个 Condition：

```java
// 队列容量
private final int capacity;
// 队列中的元素数量
private final AtomicInteger count = new AtomicInteger(0);
// 队头
private transient Node<E> head;
// 队尾
private transient Node<E> last;
// take, poll, peek 等读操作的方法需要获取到这个锁
private final ReentrantLock takeLock = new ReentrantLock();
// 如果读操作的时候队列是空的，那么等待 notEmpty 条件
private final Condition notEmpty = takeLock.newCondition();
// put, offer 等写操作的方法需要获取到这个锁
private final ReentrantLock putLock = new ReentrantLock();
// 如果写操作的时候队列是满的，那么等待 notFull 条件
private final Condition notFull = putLock.newCondition();
```

**takeLock 和 notEmpty 怎么搭配：**如果要获取（take）一个元素，需要获取 takeLock 锁，但是获取了锁还不够，如果队列此时为空，还需要队列不为空（notEmpty）这个条件（Condition）。

**putLock 需要和 notFull 搭配：**如果要插入（put）一个元素，需要获取 putLock 锁，但是获取了锁还不够，如果队列此时已满，还需要队列不是满的（notFull）这个条件（Condition）。



## SynchronousQueue

