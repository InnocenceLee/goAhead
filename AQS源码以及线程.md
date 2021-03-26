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

#### 线程状态：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190511095933572.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NzcHVkZGluZw==,size_16,color_FFFFFF,t_70)

1. NEW 新建状态，线程创建且没有执行start方法时的状态
2. RUNNABLE 可运行状态，线程已经启动，但是等待相应的资源（比如IO或者时间片切换）才能开始执行
   - RUNNABLE :当一个线程对象创建后，其他线程调用它的start()方法，该线程就进入就绪状态，Java虚拟机会为它创建方法调用栈和程序计数器。处于这个状态的线程位于可运行池中，等待获得CPU的使用权。
   - RUNNING:处于这个状态的线程占用CPU，执行程序代码。只有处于就绪状态的线程才有机会转到运行状态。
3. BLOCKED 阻塞状态，当遇到synchronized或者lock且没有取得相应的锁，就会进入这个状态
4. WAITING 等待状态，当调用`Object.wait`或者`Thread.join()`且没有设置时间，在或者`LockSupport.park`时，都会进入等待状态。
5. TIMED_WAITING 计时等待，当调用`Thread.sleep()`或者`Object.wait(xx)`或者`Thread.join(xx)`或者`LockSupport.parkNanos`或者`LockSupport.partUntil`时，进入该状态
6. TERMINATED 终止状态，线程中断或者运行结束的状态



#### FutureTask

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

#### CompletableFuture

​	一句话总结：CompletableFuture模型用于实现异步任务完成后的后置处理，因为Future本身获取任务结果的get方法是阻塞的。 CompletableFuture 提供了一些方法组合新的 Future，组合条件依赖顺序执行或并行执行。

该类实现了Future接口（ Future 在任务生产者和消费者之间的起到的桥梁作用，是一个典型通过共享内存来通信的例子，而 Go 语言的协程间通信的方式是通过 channel）和CompletionStage接口（定义了以上所有的组合条件）。

举例：

在某些业务场景中，任务处理并不是简单地执行一条 SQL， 某些长任务需要被拆解为很多小任务，而这些小任务有些可以并行处理，有些是有依赖顺序的。假设有如下一个长任务：

Task A 如下子任务：

1. 可并行处理的 Task1.1、Task1.2、Task1.3。
2. 依据第一步 Task1.1、 1.2.1.3 三个任务的结果，执行 Task2 。
3. 根据 Task2 的结果，异步的执行 Task 3.1。

TaskA 是一个比较复杂的任务，需要拆分多个子任务，其中子任务中也会涉及到子任务。如何保证这些任务的依赖关系，同时保证任务可以得到异步处理呢？

```java
 future1.thenCombine(future2， (args1， args2) -> {      ### Task 1
              println(args1);
              println(args2);
              return "3";
          }).thenApply((res) -> {                               ### Task 2
              println(res);
              return "4";
          }).thenApplyAsync((res) -> {                          ### Task 3
              println(res);
              return "5";
          });
```

future1.future2 都完成时，执行了 Combine 动作，Combine 会生成新的 Future。新的 Future 完成后将执行 thenApply， 对合并产生的结果再次处理，最后再次对结果处理，而此次处理是异步执行，即后置处理的线程和任务的消费者线程不是同一个线程。CompletableFuture 为我们提供了非常多的方法， 将其所有方法按照功能分类如下：

1. 对一个或多个 Future 合并操作，生成一个新的 Future， 参考 allOf，anyOf，runAsync， supplyAsync。
2. 为 Future 添加后置处理动作， thenAccept， thenApply， thenRun。
3. 两个人 Future 任一或全部完成时，执行后置动作：applyToEither， acceptEither， thenAcceptBothAsync， runAfterBoth，runAfterEither 等。
4. 当 Future 完成条件满足时，异步或同步执行后置处理动作： thenApplyAsync， thenRunAsync。所有异步后置处理都会添加 Async 后缀。
5. 定义 Future 的处理顺序 thenCompose 协同存在依赖关系的 Future，thenCombine。合并多个 Future的处理结果返回新的处理结果。
6. 异常处理 exceptionally ，如果任务处理过程中抛出了异常。

#### 线程关机

`kill -9 pid`相当于一次系统宕机，系统断电；`kill -15 pid`通过该命令发送一个关闭信号给jvm，可以执行一些预定的关机钩子程序。

executorService有shutdown方法和shutdownNow方法

- shutdown：ThreadPoolExecutor在shutdown之后会变成SHUTDOWN状态，无法接受新的任务，随后等待正在执行的任务执行完成。意味着，shutdown只是发出了一个命令，至于有没有关闭还是得看线程自己。
- shutdownNow：执行后状态变成STOP，并对执行中的线程调用Thread.interrupt方法（但如果线程未处理中断，则不会有任何事发生），所以并不代表“立刻关闭”。

### 几个概念

**1、并行（Parallelism）**
并行是说同一时刻做很多操作。**多进程**是实行并行的有效方法。因为它可以将许多任务分配到计算机的多个核心上。多进程很适合计算密集型的任务，因为它充分地利用了多个CPU。


**2、多进程（MultiProcessing）**
多进程是实现并行的一个有效手段。它可以充分发挥多个cpu的作用，将多个任务分配到不同的cpu上，从而实现同一时刻，处理多个任务。它很适合计算密集的任务，因为它充分利用了多个cpu。如果计算机只有一个cpu，那么多进程也是无法实现并行的。


**3、并发（Concurrency）**
并发是比并行更加宽泛的概念，它指的是，多个任务可以交叉重叠进行。用一个例子来说明下并发和并行两个概念。假设你开了一个餐馆，只有一个厨师，但同时有两桌客人点了菜。简称A桌和B桌，为了让两桌客人都满意，你可以安排厨师，交叉地为两桌客人做菜。为A桌做一道菜，再为B桌做一道菜，如此交叉进行，直到做完所有的菜。这个只能叫并发，不能叫并行。如果你多雇一个厨师，两个厨师，一个做A桌的菜，一个做B桌的菜，这个就算并行了。


**4、多线程（Threading）**
多线程是实现并发的一个手段。一个进程可以拥有多个线程。当有多个cpu时，多个线程是可以同时执行的，这时就是并行。如果只有一个cpu，那么多个线程可以交叉重叠执行，这时就是并发了。**多进程和多线程比较起来，多线程一般适用于IO密集型的任务。多进程适用于计算密集型的任务。**
可能，你会有疑问，多线程既然可以并行执行，岂不是也适用于计算密集型的任务？理论上是这样的，只是这里说多进程更适合，是说当数据量比较大时，计算任务之间没有逻辑上的依赖时，多进程更合适一些。因为每个进程都会有自己的进程内存空间，各个进程之间天然隔离。而多线程共享同一个内存空间，线程之间的同步是必须考虑的问题。而这些问题都不是计算密集型任务必须的。所以我们说计算密集型任务更适合多进程。

**5、 协程（Coroutine）**
协程是另一种实现并发的手段，这里的并发，特指不是并行的并发。当系统线程较少的时候没有什么问题，但是当线程数量非常多的时候，却产生了问题。**一是系统线程会占用非常多的内存空间，二是过多的线程切换会占用大量的系统时间。**

​	协程刚好可以解决上述2个问题。协程运行在线程之上，当一个协程执行完成后，可以选择主动让出，让另一个协程运行在当前线程之上。**协程并没有增加线程数量，只是在线程的基础之上通过分时复用的方式运行多个协程**，而且协程的切换在用户态完成(即协程不是被操作系统内核所管理，而完全是由程序所控制)，切换的代价比线程从用户态到内核态的代价小很多。

​	**协程的注意事项**

实际上协程并不是什么银弹，协程只有在等待IO的过程中才能重复利用线程，上面我们已经讲过了，线程在等待IO的过程中会陷入阻塞状态，意识到问题没有？

假设协程运行在线程之上，并且协程调用了一个阻塞IO操作，这时候会发生什么？实际上操作系统并不知道协程的存在，它只知道线程，**因此在协程调用阻塞IO操作的时候，操作系统会让线程进入阻塞状态，当前的协程和其它绑定在该线程之上的协程都会陷入阻塞而得不到调度，这往往是不能接受的。**

因此在协程中不能调用导致线程阻塞的操作。也就是说，协程只有和异步IO结合起来，才能发挥最大的威力。

那么如何处理在协程中调用阻塞IO的操作呢？一般有2种处理方式：

1. **在调用阻塞IO操作的时候，重新启动一个线程去执行这个操作，等执行完成后，协程再去读取结果。这其实和多线程没有太大区别。**
2. **对系统的IO进行封装，改成异步调用的方式，这需要大量的工作，最好寄希望于编程语言原生支持。**

协程对计算密集型的任务也没有太大的好处，计算密集型的任务本身不需要大量的线程切换，因此协程的作用也十分有限，反而还增加了协程切换的开销。

**协程只有和异步IO结合起来才能发挥出最大的威力。**

总结：

**在所有线程相互独立且不会阻塞的模式下，抢断式的线程调度器是不错的选择**。因为它可以保证所有的线程都可以被分到时间片不被程序员的垃圾代码所累。这对于某些事情来说是至关重要的，例如计时器、回调、IO触发器（譬如说处理请求）什么的。

但是在线程不是相互独立，经常因为争抢而阻塞的情况下，抢断式的线程调度器就显得脱了裤子放屁了，**既然你们只能一个个的跑，那抢断还有什么意义？**让你们自己去让出时间片就好了，不然还要上下文切换开销。

再往后，**大家发现经常有阻塞的情况下**，主动让出时间片的协程模式比抢占式分配的效率要好，也简单得多。

