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



### 关于线程

线程状态：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190511095933572.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NzcHVkZGluZw==,size_16,color_FFFFFF,t_70)

1. NEW 新建状态，线程创建且没有执行start方法时的状态
2. RUNNABLE 可运行状态，线程已经启动，但是等待相应的资源（比如IO或者时间片切换）才能开始执行
   - RUNNABLE :当一个线程对象创建后，其他线程调用它的start()方法，该线程就进入就绪状态，Java虚拟机会为它创建方法调用栈和程序计数器。处于这个状态的线程位于可运行池中，等待获得CPU的使用权。
   - RUNNING:处于这个状态的线程占用CPU，执行程序代码。只有处于就绪状态的线程才有机会转到运行状态。
3. BLOCKED 阻塞状态，当遇到synchronized或者lock且没有取得相应的锁，就会进入这个状态
4. WAITING 等待状态，当调用`Object.wait`或者`Thread.join()`且没有设置时间，在或者`LockSupport.park`时，都会进入等待状态。
5. TIMED_WAITING 计时等待，当调用`Thread.sleep()`或者`Object.wait(xx)`或者`Thread.join(xx)`或者`LockSupport.parkNanos`或者`LockSupport.partUntil`时，进入该状态
6. TERMINATED 终止状态，线程中断或者运行结束的状态



> FutureTask最主要特性：**相同的FutureTask对象，只会被执行一次，来保证任务的唯一性，且线程安全。** 

FutureTask的get方法源码解析：

```java
/**
*state值表示状态：
	private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
* 即当状态大于等于NORMAL时，直接返回，不会再执行任务
*/
public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

	/**
		outcome存储任务运行结果
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }

//上述执行一次的原因在run方法执行后，会set(result)会将state改成NORMAL
public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

	protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```

