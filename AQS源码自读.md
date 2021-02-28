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

