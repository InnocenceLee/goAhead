java8中HashMap使用链表法避免哈希冲突（相同hash值），当链表长度大于TREEIFY_THRESHOLD（默认为8）时，将链表转换为红黑树。当小于等于UNTREEIFY_THRESHOLD（默认为6）时，又会退化回链表以达到性能均衡。 下图为HashMap的数据结构（数组+链表+红黑树 ）



![img](https:////upload-images.jianshu.io/upload_images/7368936-0424df309cb12e86.png?imageMogr2/auto-orient/strip|imageView2/2/w/552/format/webp)

> Hashtable和ConcurrentHashMap两者均不允许key和value为null，否则会报错空指针异常。对于Hashtable主要是设计当时想key都能实现hashCode和equals方法。并且ConcurrentHashmap和Hashtable都是支持并发的，这样会有一个问题，**当你通过get(k)获取对应的value时，如果获取到的是null时，你无法判断，它是put（k,v）的时候value为null，还是这个key从来没有做过映射，此时由于是并发，所以你也无法通过contain判断**。HashMap是非并发的，可以通过contains(key)来做这个判断。而支持并发的Map在调用m.contains（key）和m.get(key),m可能已经不同了。
>

## HashMap在并发时出现的问题

### 1.多线程put的时候可能导致元素丢失

jdk7主要问题出在addEntry方法的new Entry (hash, key, value, e)，如果两个线程都同时取得了e,则他们下一个元素都是e，然后赋值给table元素的时候有一个成功有一个丢失。

jdk8中如果没有hash碰撞则会直接插入元素。如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第6行代码中。假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给**覆盖**，发生线程不安全。

### 2.多线程put后可能导致get死循环

造成死循环的原因是多线程进行put操作时，触发了HashMap的扩容（resize函数），出现链表的两个结点形成闭环，导致死循环。

JDK7的put操作头插法（并发会出现死循环）、jdk8变成尾插法（避免循环链）

## jdk7单线程resize过程



![img](https:////upload-images.jianshu.io/upload_images/7368936-14e7d64844d09c04.png?imageMogr2/auto-orient/strip|imageView2/2/w/653/format/webp)

HashMap的Resize过程

首先我们把resize函数中的transfer()的关键代码贴出来：

```ruby
while(null != e) {
    Entry<K,V> next = e.next;
    if (rehash) {
        e.hash = null == e.key ? 0 : hash(e.key);
    }
    int i = indexFor(e.hash, newCapacity);
    e.next = newTable[i];
    newTable[i] = e;  //newTable[i]可以理解成新链表的头指针
    e = next;
}
```

我们可以再简化一下，因为中间的i就是判断新表的位置，我们可以跳过。简化后代码：

```ruby
while(null != e) {
    Entry<K,V> next = e.next;
    e.next = newTable[i]; //假设线程一执行完这里就被调度挂起了
    newTable[i] = e;
    e = next;
}
```

去掉了一些与本过程冗余的代码，意思就非常清晰了：

1. Entry<K,V> next = e.next;// 因为是单链表，如果要转移头指针，一定要保存下一个结点，不然转移后链表就丢了
2. e.next = newTable[i];// e要插入到链表的头部，所以要先用e.next指向新的 Hash表第一个元素（为什么不加到新链表最后？因为复杂度是 O（N））
3. newTable[i] = e;// 现在新Hash表的头指针仍然指向e没转移前的第一个元素，所以需要将新Hash表的头指针指向e
4. e = next;// 转移e的下一个结点

### JDK7并发时resize过程

1. 假设我们线程一执行完`e.next = newTable[i];`就被调度挂起了



![img](https:////upload-images.jianshu.io/upload_images/7368936-ae265aad2045fda6.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/616/format/webp)

并发的resize过程

而我们的线程二执行完成了。于是我们有下面的这个样子。

> 因为Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。我们可以看到链表的顺序被反转后。

1. 线程一被调度回来执行。 
   1. 执行 newTalbe[i] = e。
   2. 执行e = next，导致了e指向了key(7)。
   3. 下一次循环的next = e.next导致了next指向了key(3)。



![img](https:////upload-images.jianshu.io/upload_images/7368936-8e5ac468e4bec4ad.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/591/format/webp)

并发的resize过程

1. 一切安好。
    线程一接着工作。把key(7)摘下来，放到newTable[i]的第一个，然后把e和next往下移。



![img](https:////upload-images.jianshu.io/upload_images/7368936-868abdc30bad2f6c.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/627/format/webp)

并发的resize过程

1. 环形链接出现。
    e.next = newTable[i] 导致 key(3).next 指向了 key(7)。注意：此时的key(7).next 已经指向了key(3)， 环形链表就这样出现了。



![img](https:////upload-images.jianshu.io/upload_images/7368936-19bc623b4fecc8a0.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/623/format/webp)

并发的resize过程

于是，当我们的线程一调用到，HashTable.get(11)时，悲剧就出现了——Infinite Loop。

## JDK8的构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity); //jdk1.8阈值为传入容量的最近2次幂
    }
 public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

```

## JDK8的hash算法

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }//相当简单，就是key的hashcode然后高位右移减少哈希碰撞

//计算桶下标，由于n是2次幂，所以&运算代替%
i = (n - 1) & hash
```



## JDK8的put源码

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//如果当前Node数组为空或者长度为0，则进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//获取当前hash位置节点p，如果为空，则直接将新元素放置于此处
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
			//如果当前hash位置节点p不为空，但是key与新增的元素key一致，则进行value的覆盖
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
			//如果p不为空，且key也不相等，但此时节点p是TreeNode类型，则调用putTreeVal方法新增
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
			//如果p不为空，且key也不相等，此时节点p不是TreeNode类型（即是Node类型），则将新节点添加到该位置所在链表的最后的一个元素
			//并且添加完成后，如果此时该位置的链表超过了阈值，则转化为红黑树，需要注意treeifyBin内部判断如果元素个数小于64，则会扩容，而不是树化
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
			
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
				//LinkedHashMap中可配置LRU即根据访问顺序（accessOrder）将被访问节点移动到末尾节点
                afterNodeAccess(e);
                return oldValue;
            }
        }
		//修改次数增加
        ++modCount;
		//如果此时size大于阈值，则进行扩容
        if (++size > threshold)
            resize();
		//根据LinkedHashMap中重写的removeEldestEntry方法（是否移除最久未使用的键）进行
        afterNodeInsertion(evict);
        return null;
    }
//转换为红黑树方法，小于64时依然会扩容而不是树化
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) //MIN_TREEIFY_CAPACITY值为64
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

## JDK8的resize源码

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 当原长度大于16时，新数组的长度和阈值都扩展为原数组的2倍
        }//oldThr大于0说明构造时传入了初始容量
        else if (oldThr > 0) // initial capacity was placed in threshold（初始容量放于此）
            newCap = oldThr;//如果老数组为空但是阈值大于0，则将新数组长度等于阈值
        else {               // 则第一次全部初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {//若为0，则重新计算阈值
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {//遍历旧Node数组
                Node<K,V> e;//用于暂存每个Node
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)//如果当前Node没有后续链表节点，则直接定位赋值到新数组
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//当前Node有后续节点且类型为红黑树，下详情
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { //当前Node有后续节点且类型为链表
                        //loHead和loTail表示新增的一bit位&结果为0的，即不需要迁移位置的节点链表
                        Node<K,V> loHead = null, loTail = null;
                        //hiHead和hiTail表示新增的一bit位&结果为1的，即需要变更的位置为当前index+oldSize处的
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //e.hash & oldCal(16)的值就是最高1位随机为0或1，为0就在原位
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {//最高位为1需要迁移的
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //开始给新节点赋值
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // 将红黑树重新拆解为两个链表，lo表示扩容后在原位置，hi表示扩容后位置变化，并且保留节点顺序
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;//lc和hc表示两个链表的长度
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {//原位置
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {//不在原位置
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {//当loHead不为空，处理loHead
                if (lc <= UNTREEIFY_THRESHOLD)//且长度小于等于链表化的阈值
                    tab[index] = loHead.untreeify(map);//链表化
                else {
                    tab[index] = loHead;// 原本就是树，直接赋值
                    if (hiHead != null) // 但是如果hiHead不为空，则当前树跟原有树有变化，需重新树化以达到平衡
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {//当hiHead不为空，处理hiHead
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

jdk8创建方法：

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);//设置为最近的2的整数次幂的数。
    }
```

jdk8中传不传容量都会在构造方法中设置负载因子为默认0.75（若传值会设置threhold阈值，否则为0），然后在put时候如果size为空则进行容量和阈值的初始化。并且当每次put完后都会判断当前map长度是否大于threhold，如果大于并且原长度大于16，此时容量和阈值纷纷2倍。但如果原长度未大于16，则只需容器2倍，然后按照负载因子（可能是人为指定的）计算出thres阈值。

创建时候oldCap都为0，在第一次resize时候才设置newCap值为oldThres，并且此时会重新计算threshold值。

而在jdk7中，初始threshold值为传入的cap值，在第一次put时cap会设置为thres最近的2幂次值，然后threshold重新计算，并且Entry数组直接初始化为cap大小！（jdk8的数组不会初始化大小，而是插入一个后判断resize()）。

```java
//jdk8返回大于输入参数且最近的2的整数次幂的数。
//先来分析有关n位操作部分：先来假设n的二进制为01xxx...xxx。接着
//对n右移1位：001xx...xxx，再位或：011xx...xxx
//对n右移2为：00011...xxx，再位或：01111...xxx
//此时前面已经有四个1了，再右移4位且位或可得8个1
//同理，有8个1，右移8位肯定会让后八位也为1。
//综上可得，该算法让最高位的1后面的位全变为1。最后再让结果n+1，即得到了2的整数次幂的值了。
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

//jdk7中计算方式如下：可见与8的区别在于对于数据的处理。jdk8是减一，jdk7是...
private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }
public static int highestOneBit(int i) {
        // HD, Figure 3-1
        i |= (i >>  1);
        i |= (i >>  2);
        i |= (i >>  4);
        i |= (i >>  8);
        i |= (i >> 16);
        return i - (i >>> 1);
    }
```

## jdk8的put过程图解

![img](https:////upload-images.jianshu.io/upload_images/7368936-f6436cc54ff9ba42.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

## JDK7的构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        this.loadFactor = loadFactor;
        threshold = initialCapacity; //与1.8不同的是1.7中threhold初始设置为传入的值（首次put时infalteTable方法才会更新阈值），而1.8中是已经处理成了最近的2次幂。
        init();
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }
```

## JDK7的hash算法

```java
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h ^= k.hashCode();
        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```



## JDK7的put、resize源码

```java
public V put(K key, V value) {
    	//空表则初始化，初始化capicity为阈值的2的幂次值，并初始化空Entry数组
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)//key为空直接hash为0处插入
            return putForNullKey(value);
        int hash = hash(key);//计算hash
        int i = indexFor(hash, table.length);//计算entry数组中位置：hash&(n-1)
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;//遍历hash对应的数组位置处的链表，如果发现key相同则进行覆盖
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;//增加修改次数，普通transient变量
        addEntry(hash, key, value, i);//进入节点插入方法
        return null;
    }

	void addEntry(int hash, K key, V value, int bucketIndex) {
        //第一步首先判断size是否达到阈值并且待插入位置有元素即产生hash碰撞，两个条件满足后进行扩容。
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);//如果发生扩容，则重新计算位置
        }
        //若不满足扩容则直接头插法插入节点
        createEntry(hash, key, value, bucketIndex);
    }
	//直接头插法，e为原始链表头元素，new Entry方法传入e，会完成this.next=e的操作
	void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

	//扩容方法
	void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        Entry[] newTable = new Entry[newCapacity];
        //关键是此处的transfer方法
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
	
	void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {//table是
            while(null != e) {
                Entry<K,V> next = e.next;//此处如果多线程扩容，线程A执行完此处阻塞，线程B扩容完毕，此时线程A再继续扩容，就可能会导致循环链表
                if (rehash) {//如果需要重新计算hash，则重新计算hash
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                //从这个位置可以看到新数组扩容迁移元素采用头插法，**会引起原链表倒序**
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```



## JDK8中HashMap的优化

### 1.长链表的优化

在JDK8中，如果链表的长度大于等于8 ，那么链表将转化为红黑树；当长度小于等于6时，又会变成一个链表

> 红黑树的平均查找长度是log(n)，长度为8的时候，平均查找长度为3，如果继续使用链表，平均查找长度为8/2=4，这才有转换为树的必要。链表长度如果是小于等于6，6/2=3，虽然速度也很快的，但是转化为树结构和生成树的时间并不会太短。
>  还有选择6和8，中间有个差值7可以有效防止链表和树频繁转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。

> **注意，当数组长度小于64（常量定义）的时候，扩张数组长度一倍，否则的话把链表转为树。并不是每次都直接树化的。**MIN_TREEIFY_CAPACITY 最小树形化容量阈值：即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）否则，若桶内元素太多时，则直接扩容，而不是树形化！

此外，在JDK7中，hash计算的时候会对操作数进行右移操作，计算复杂，目的是将高位也参与运算，减少hash碰撞；在JDK8中，链表可以转变成红黑树，所以hash计算也变得简单。下面的图为JDK8中的hash计算和索引计算。



![img](https:////upload-images.jianshu.io/upload_images/7368936-16e941c679177719.png?imageMogr2/auto-orient/strip|imageView2/2/w/1198/format/webp)

### 2.hash碰撞的插入方式的优化

发生hash碰撞时，JDK7会在链表头部插入，而JDK8会在链表尾部插入。头插法是操作速度最快的，找到数组位置就直接找到插入位置了，但JDK8之前hashmap这种插入方法在并发场景下会出现死循环。JDK8开始hashmap链表在节点长度达到8之后会变成红黑树，这样一来在数组后节点长度不断增加时，遍历一次的次数就会少很多（否则每次要遍历所有），而且也可以避免之前的循环列表问题。同时如果变成红黑树，也不可能做头插法了。

### 3.扩容机制的优化

在JDK7中，对所有链表进行rehash计算；在JDK8中，实际上也是通过取余找到元素所在的数组的位置，取余的方式在putVal里面：`i = (n - 1) & hash`。我们假设，在扩容之前，key取余之后留下了n位。扩容之后，容量变为2倍，所以key取余得到的二进制数就有n+1位。在这n+1位里面，如果第1位是0，那么扩容前后这个key的位置还是在相同的位置（因为hash相同，并且余数的第1位是0，和之前n位的时候一样，所以余数还是一样，位置就一样了）；如果这n+1位的第一位是1，那么就和之前的不同，那么这个key就应该放在之前的位置再加上之前整个数组的长度的位置。这样子就减少了移动所有数据带来的消耗。（慢慢读两遍，想明白了，就觉得这个其实不看图更好理解）



![img](https:////upload-images.jianshu.io/upload_images/7368936-28489dc0a1d001a9.png?imageMogr2/auto-orient/strip|imageView2/2/w/890/format/webp)

JDK8的rehash计算

## 常见FAQ

### HashMap扩容条件

查看JDK7源码的put函数，然后跳转到addEntry函数。然后你会发现JDK7中hashmap扩容是要同时满足两个条件：

1. 当前数据存储的数量（即size()）大小必须大于等于阈值
2. 当前加入的数据是否发生了hash冲突

因为上面这两个条件，所以存在下面两种情况：

1. 就是hashmap在存值的时候（默认大小为16，负载因子0.75，阈值12），可能达到最后存满16个值的时候，再存入第17个值才会发生扩容现象，因为前16个值，每个值在底层数组中分别占据一个位置，并没有发生hash碰撞。
2. 当然也有可能存储更多值（超多16个值，最多可以存26个值）都还没有扩容。原理：前11个值全部hash碰撞，存到数组的同一个位置（这时元素个数小于阈值12，不会扩容），后面所有存入的15个值全部分散到数组剩下的15个位置（这时元素个数大于等于阈值，但是每次存入的元素并没有发生hash碰撞，所以不会扩容），前面11+15=26，所以在存入第27个值的时候才同时满足上面两个条件，这时候才会发生扩容现象。

**而在JDK8中，扩容的条件只有一个，就是当前容量大于阈值（阈值等于当前hashmap最大容量乘以负载因子），并且在每次插入完成后会判断是否需要扩容（jdk7是扩容后再进行插入）**

### HashMap在JDK7扩容过程中计算新索引的方法

转移操作 = 按旧链表的正序遍历链表、在新链表的头部依次插入，即在转移数据、扩容后，容易出现链表逆序的情况 。

通过transfer方法将旧数组中的元素复制到新数组，在这个方法中进行了包括释放旧的Entry中的对象引用，该过程中如果需要重新计算hash值就重新计算，然后根据indexfor（）方法计算索引值。而索引值的计算方法为: `h & (length-1)`
 ，即hashcode计算出的hash值和数组长度进行与运算。

> 一般认为，Java的%、/操作比&慢10倍左右，因此采用&运算而不是`h % length`会提高性能。

### HashMap在JDK8扩容过程中计算索引的方法

**这个设计确实非常的巧妙，既省去了重新计算hash值的时间，也就是说1.8不用重新计算hash值而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。**这一块就是JDK1.8新增的优化点。有一点注意区别，**JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，因为他采用的是头插法，先拿出旧链表头元素。但是从下图可以看出，JDK1.8不会倒置，采用的尾插法。**



![img](https:////upload-images.jianshu.io/upload_images/7368936-55c84ca5d0c034dc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

JDK8的rehash计算

### 为什么 HashMap中 String、Integer 这样的包装类适合作为 key键



![img](https:////upload-images.jianshu.io/upload_images/7368936-f19db1204322e6c0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Key的选择

### HashMap中的key如果是Object类型，则需实现哪些方法？



![img](https:////upload-images.jianshu.io/upload_images/7368936-ae3088f95933c3f9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

HashMap的key和value都可以是Null，虽然无法hash，但是程序作了处理，默认哈希值为0。

## ConcurrentHashMap

参考链接：<https://blog.csdn.net/ZOKEKAI/article/details/90051567>

#### **JDK1.7版本的ConcurrentHashMap的实现原理**

在JDK1.7中ConcurrentHashMap采用了**数组+Segment+分段锁**的方式实现。

**1.Segment(分段锁)**

ConcurrentHashMap中的**分段锁称为Segment**，它即类似于HashMap的结构，即内部拥有一个Entry数组，数组中的每个元素又是一个链表,同时又是一个ReentrantLock（Segment继承了ReentrantLock）。

 **2.内部结构**

ConcurrentHashMap使用分段锁技术，将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问，能够实现真正的并发访问。

面的结构我们可以了解到，ConcurrentHashMap定位一个元素的过程需要进行两次Hash操作。

**第一次Hash定位到Segment，第二次Hash定位到元素所在的链表的头部。**

 **3.该结构的优劣势**

**坏处**

这一种结构的带来的副作用是Hash的过程要比普通的HashMap要长

**好处**

写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作（刚好这些写操作都非常平均地分布在所有的Segment上）。

所以，通过这一种结构，ConcurrentHashMap的并发能力可以大大的提高。

#### JDK1.8版本的ConcurrentHashMap的实现原理

JDK8中ConcurrentHashMap参考了JDK8 HashMap的实现，采用了**数组+链表+红黑树**的实现方式来设计。

**JDK8中彻底放弃了Segment转而采用的是Node，其设计思想也不再是JDK1.7中的分段锁思想。**

**Node：保存key，value及key的hash值的数据结构。其中value和next都用volatile修饰，保证并发的可见性。**

**Java8 ConcurrentHashMap结构基本上和Java8的HashMap一样，不过保证线程安全性。**

#### ConcurrentHashMap总结

其实可以看出JDK1.8版本的ConcurrentHashMap的数据结构已经接近HashMap，相对而言，ConcurrentHashMap只是增加了同步的操作来控制并发，从JDK1.7版本的ReentrantLock+Segment+HashEntry，到JDK1.8版本中synchronized+CAS+HashEntry+红黑树。

**1.数据结构**：取消了Segment分段锁的数据结构，取而代之的是数组+链表+红黑树的结构。

**2.保证线程安全机制**：JDK1.7采用segment的分段锁机制实现线程安全，其中segment继承自ReentrantLock。JDK1.8采用CAS+Synchronized保证线程安全。

**3.锁的粒度**：原来是对需要进行数据操作的Segment加锁，现调整为对每个数组元素加锁（Node）。

**4.链表转化为红黑树**:定位结点的hash算法简化会带来弊端,Hash冲突加剧,因此在链表节点数量大于8时，会将链表转化为红黑树进行存储。

**5.查询时间复杂度**：从原来的遍历链表O(n)，变成遍历红黑树O(logN)。

#### ConcurrentHashMap扩容

### 什么情况会触发扩容

当往hashMap中成功插入一个key/value节点时，有可能触发扩容动作：
1、如果新增节点之后，所在链表的元素个数达到了阈值 8，则会调用`treeifyBin`方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，如果数组长度n小于阈值`MIN_TREEIFY_CAPACITY`，默认是64，则会调用`tryPresize`方法把数组长度扩大到原来的两倍，并触发`transfer`方法，重新调整节点的位置。

2、新增节点之后，会调用`addCount`方法记录元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，触发扩容 。

3、 扩容状态下其他线程对集合进行插入、修改、删除、合并、compute 等操作时遇到 ForwardingNode 节点会触发扩容 。

注意：桶上链表长度达到 8 个或者以上，并且数组长度为 64 以下时只会触发扩容而不会将链表转为红黑树 。

 **CPU核数与迁移任务hash桶数量分配的关系**

**![img](https://img-blog.csdnimg.cn/20190510093016265.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

 

 

**单线程下线程的任务分配与迁移操作**

**![img](https://img-blog.csdnimg.cn/20190510093338545.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

 

 

 

**多线程如何分配任务？**

**![img](https://img-blog.csdnimg.cn/20190510093435247.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

 

 

 

**普通链表如何迁移？**

**![img](https://img-blog.csdnimg.cn/20190510093520878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

 

 

*什么是 lastRun 节点？*

**![img](https://img-blog.csdnimg.cn/20190510093610941.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

**hash桶迁移中以及迁移后如何处理存取请求？**

![img](https://img-blog.csdnimg.cn/20190602184408984.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)

 

 

 

**多线程迁移任务完成后的操作**

**![img](https://img-blog.csdnimg.cn/20190510093843462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

**![img](https://img-blog.csdnimg.cn/20190510224608540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pPS0VLQUk=,size_16,color_FFFFFF,t_70)**

ConcurrentHashMap的sizeCtl、lastRun、hn、ln