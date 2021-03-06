# BlockingQueue阻塞队列

> 最大的特点是：支持生产者-消费者模型。

首先要区分三个概念：
> 其实就是搞清楚“同步”和“阻塞”是两个概念

1. 队列（支持FIFO）
2. 同步队列（并发控制，在队列的基础上，插入与删除保证同步）
3. 阻塞队列（在同步队列的基础上，支持阻塞的插入与删除）
   1. 支持阻塞的插入：当队列满时，插入元素的线程会被阻塞住，直到队列不满。
   2. 支持阻塞的移除：当队列空时，获取元素的线程会被阻塞住，直到队列不空。

| 方法 | 抛出异常 | 返回特殊值 | 一直阻塞 | 超时退出 |
| :--- | :--- | :--- | :--- | :--- |
| 插入 | add（e） | offer（e） | put（e） | offer（e,time,unit） |
| 移除 | remove（） | poll（） | take（） | poll（time,unit） |
| 检查 | element（） | peek（） | 不可用 | 不可用 |

Java中的7个阻塞队列

| 阻塞队列 | 底层数据结构 | 有界/无界 | 特性 |
| :--- | :--- | :--- | :--- |
| ArrayBlockingQueue | 数组 | 有界，初始化传入 |  |
| LinkedBlockingQueue | 链表 | 有界，默认MAX\_INT |  |
| PriorityBlockingQueue | 堆（数组） | 无界，自增长 | 支持元素排序 |
| DelayQueue | PriorityQueue（数组） | 无界，自增长 | 当元素的延迟期满后，才能从队列中取出元素 |
| SynchronousQueue |  |  |  |
| LinkedTransferQueue | 链表 | 无界 |  |
| LinkedBlockingDeque | 双向链表 |  | 双向队列，理论上减少一半的竞争，可用于“工作窃取” |

DelayQueue：元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。

* 基于DelayQueue的缓存系统：元素自定有效期，另外使用一个线程循环查询DelayQueue，一旦能查出来，则表示缓存到期了。
* 基于DelayQueue的定时任务调度：将周期任务放入DelayQueue中，一旦从队列中获取到元素，则执行任务（定时任务一般用 ScheduledThreadPoolExecutor）

# 阻塞队列两大问题：同步 + 阻塞

以ArrayBlockingQueue的put方法为例
```
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock; // 同步控制
        lock.lockInterruptibly();
        try {
            while (count == items.length) // 阻塞控制
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```


### 同步的实现原理（AQS）
ReentrantLock

ReentrantLock的实现参见[AbstractQueuedSynchronizer](/tong-bu-kuang-jia/abstractqueuedsynchronizer.md)

### 阻塞的实现原理（自旋+等待通知）

> while 自旋 + 等待通知模式

例如：ArrayBlockingQueue使用Condition

```
// notEmpty = lock.newCondition();
// notFull =  lock.newCondition();

// 阻塞式插入
while (count == items.length)
    notFull.await(); // 若满了，则等待队列不满notFull；若被唤醒后，队列还是满的，则继续等待（while循环）
enqueue(e);
```


入队列后，通知等待notEmpty的线程
```
private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
```



# 多线程环境下的通知模式是什么？

参见[Java线程间通信](/jvm/javaxian-cheng-jian-tong-xin.md)

