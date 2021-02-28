### 关于AQS

**AbstractQueuedSynchronizer**有一个Node内部类，Node属性有以下：

- thread：该节点对应的线程对象
- pre：该节点的前置节点
- next：该节点的后置节点
- nextWaiter：记录下一个在condition上等待的节点
- waitStatus：节点状态。取值分别有0（初始化值）、1（CANCELED、表示已取消）、-1（SINGINAL、表示后继节点的线程需要被唤醒unpark）、-2（CONDITION、表示当前线程正在一个condition上等待）、-3（indicate the next acquireShared should unconditionally propagate）

**AbstractQueuedSynchronizer**的构成：

属性：

- **一个状态state字段**：同步状态，获取锁则加1，释放锁则减1
- **一个同步队列（Node类型的head、tail属性）**
- 自旋超时阈值：默认为1000毫秒
- Unsafe类对各个属性的cas操作：注意，在jdk10中已经换成varHandle操作

方法：

- 核心的tryAcquire和tryRelease方法直接抛错，强制子类必须实现
- 自身已实现accquire和release方法

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

当使用`Condition`的时候，等待队列的概念就出来了。Condition的获取一般都要与一个锁Lock相关，一个锁上面可以生产多个Condition。

Condition接口的主要实现类是AQS的内部类`ConditionObject`，**每个Condition对象都包含一个等待队列**。该队列是Condition对象实现等待/通知的关键。关系图如下：

![](E:\study\再出发\image\aqs.jpg)

- 等待

  调用condition的await方法，将会使当前线程进入等待队列并释放锁(先加入等待队列再释放锁)，同时线程状态转为等待状态。

  从同步队列和阻塞队列的角度看，调用await方法时，相当于**同步队列的首节点移到condition的等待队列中**

- 通知

  **调用condition的signal方法时，将会把等待队列的首节点移到等待队列的尾部，然后唤醒该节点。被唤醒，并不代表就会从await方法返回，也不代表该节点的线程能获取到锁，它一样需要加入到锁的竞争acquireQueued方法中去，只有成功竞争到锁，才能从await方法返回。**

![](E:\study\再出发\image\lock.jpg)

### 关于CountDowanLatch和CyclicBarrier

超级经典：https://blog.csdn.net/qq_39241239/article/details/87030142