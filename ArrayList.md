#### ArrayList

ArrayList就是数组列表，主要用来装载数据，当我们装载的是基本类型的数据int，long，boolean，short，byte…的时候我们只能存储他们对应的包装类，它的主要底层实现是数组Object[] elementData。

​	通过无参构造方法的方式ArrayList()初始化，则赋值底层数Object[] elementData为一个默认空数组Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}所以数组容量为0，只有真正对数据进行添加add时，才分配默认DEFAULT_CAPACITY = 10的初始容量。

```java
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

> 数组的长度是有限制的，而ArrayList是可以存放任意数量对象，长度不受限制，那么他是怎么实现的呢？

其实实现方式比较简单，他就是通过数组扩容的方式去实现的。

就比如我们现在有一个长度为10的数组，现在我们要新增一个元素，发现已经满了，那ArrayList会怎么做呢？

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzW9kGwS65sOQzl9X9ZBsK5sj4NCXfMoNg7f9MCDm5xSI3UkuPR7T0ZuRUcAlKkcoL50N4iaia1c5WA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

第一步他会重新定义一个长度为10+10/2的数组也就是新增一个长度为15的数组。



新增的时候：

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 这里校验是否需要扩容，如果需要则会先自增modCount值，然后调用grow方法
        elementData[size++] = e;
        return true;
  }
```

在扩容的时候，老版本的jdk和8以后的版本是有区别的，8之后的效率更高了，采用了位运算，**右移**一位，其实就是除以2这个操作。

```java

private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); //在这里！！！
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

指定位置插入：

```java
public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);  // 这里校验是否需要扩容，如果需要则会先自增modCount值，然后调用grow方法
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

不知道大家看懂**arraycopy**的代码没有，我画个图解释下，你可能就明白一点：

比如有下面这样一个数组我需要在index 5的位置去新增一个元素A

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzW9kGwS65sOQzl9X9ZBsK5ZSuaic23wv1eSiamttx3zIs0Tu4QQJJoVhdwWfCh34Ppg6nFSS0CBicJQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那从代码里面我们可以看到，他复制了一个数组，是从index 5的位置开始的，然后把它放在了index 5+1的位置

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzW9kGwS65sOQzl9X9ZBsK5b2RZbr12SLN6dXFibAx3u6aZdk34jH6WKWMm83yT6PotUdDW4mEZRWA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

给我们要新增的元素腾出了位置，然后在index的位置放入元素A就完成了新增的操作了

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzW9kGwS65sOQzl9X9ZBsK57rRg4F3JEcpM310I1owAnRGtMRv5yvWYxvCicVNPNWicPBzCvSxbZYibA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

至于为啥说他效率低，我想我不说你也应该知道了，我这只是在一个这么小的List里面操作，要是我去一个几百几千几万大小的List新增一个元素，那就需要后面所有的元素都复制，然后如果再涉及到扩容啥的就更慢了不是嘛。





> ArrayList的默认数组大小为什么是10？

其实我也没找到具体原因。

据说是因为sun的程序员对一系列广泛使用的程序代码进行了调研，结果就是10这个长度的数组是最常用的最有效率的。也有说就是随便起的一个数字，8个12个都没什么区别，只是因为10这个数组比较的圆满而已。

> **ArrayList（int initialCapacity）会不会初始化数组大小？**

**不会初始化数组大小！**

而且将构造函数与initialCapacity结合使用，然后使用set（）会抛出异常，尽管该数组已创建，但是大小设置不正确。

使用sureCapacity（）也不起作用，因为它基于elementData数组而不是大小。

还有其他副作用，这是因为带有sureCapacity（）的静态DEFAULT_CAPACITY。

直接操作一下代码，大家会发现我们虽然对ArrayList设置了初始大小，但是我们打印List大小的时候还是0，我们操作下标set值的时候也会报错，数组下标越界。

再结合源码，大家仔细品读一下，这是Java Bug里面的一个经典问题了，还是很有意思的，大家平时可能也不会注意这个点。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1FpzW9kGwS65sOQzl9X9ZBsK5II4Yyibqv9dWD9HGd8XTK6IekJ25sHfLels5aib3nuKAdtgJPxiadOPAw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> ArrayList插入删除一定慢么？

取决于你删除的元素离数组末端有多远，ArrayList拿来作为堆栈来用还是挺合适的，push和pop操作完全不涉及数据移动操作。

> 那他的删除怎么实现的呢？

删除其实跟新增是一样的，不过叫是叫删除，但是在代码里面我们发现，他还是在copy一个数组。

> 为啥是copy数组呢？

```java
public E remove(int index) {
        rangeCheck(index);
        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

> ArrayList是线程安全的么？

当然不是，线程安全版本的数组容器是Vector。

Vector的实现很简单，就是把所有的方法统统加上synchronized就完事了。

你也可以不使用Vector，用Collections.synchronizedList把一个普通ArrayList包装成一个线程安全版本的数组容器也可以，原理同Vector是一样的，就是给所有的方法套上一层synchronized。

> ArrayList用来做队列合适么？

队列一般是FIFO（先入先出）的，如果用ArrayList做队列，就需要在数组尾部追加数据，数组头部删除数组，反过来也可以。

但是无论如何总会有一个操作会涉及到数组的数据搬迁，这个是比较耗费性能的。

**结论**：ArrayList不适合做队列。

> 那数组适合用来做队列么？

数组是非常合适的。

比如ArrayBlockingQueue内部实现就是一个环形队列，它是一个定长队列，内部是用一个定长数组来实现的。

另外著名的Disruptor开源Library也是用环形数组来实现的超高性能队列，具体原理不做解释，比较复杂。

**简单点说就是使用两个偏移量来标记数组的读位置和写位置，如果超过长度就折回到数组开头，前提是它们是定长数组。**

> ArrayList的遍历和LinkedList遍历性能比较如何？

论遍历ArrayList要比LinkedList快得多，ArrayList遍历最大的优势在于内存的连续性，CPU的内部缓存结构会缓存连续的内存片段，可以大幅降低读取内存的性能开销。

> 看完构造方法、添加方法、扩容方法之后，上文第1个问题终于有了答案。原来，`new ArrayList()`会将`elementData` 赋值为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，`new ArrayList(0)`会将`elementData` 赋值为 EMPTY_ELEMENTDATA，EMPTY_ELEMENTDATA添加元素会扩容到容量为`1`，而DEFAULTCAPACITY_EMPTY_ELEMENTDATA扩容之后容量为`10`。

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        ArrayList<Integer> zeroList = new ArrayList<>(0);
        System.out.println("初始为0时，长度为" + getCapacity(zeroList));//输出0
        zeroList.add(1);
        System.out.println("初始为0时，长度为" + getCapacity(zeroList));//输出1

        ArrayList<Integer> defaultList = new ArrayList<>();
        System.out.println("默认时，长度为" + getCapacity(defaultList));//输出0
        defaultList.add(1);
        System.out.println("默认时，长度为" + getCapacity(defaultList));//输出10
        System.out.println("默认时，打印第10个位置依然会报错，因为size是独立的");//数组越界异常
    }

    public static int getCapacity(ArrayList list) throws NoSuchFieldException, IllegalAccessException {
        Class<ArrayList> listClass = ArrayList.class;
        Field elementData = listClass.getDeclaredField("elementData");
        elementData.setAccessible(true);
        Object[] obj = (Object[]) elementData.get(list);
        return obj.length;

    }
```



## 总结

1. ArrayList有两个空数组属性，DEFAULTCAPACITY_EMPTY_ELEMENTDATA和EMPTY_ELEMENTDATA。
2. ArrayList默认长度是10，但是如果创建时指定容量，此时size()依然为0，add(i, value)会报错。因为size()方法和add指定下标时都检查的size属性字段，该字段是实时的。
3. ArrayList适合作为堆栈，因为push和pop都不会涉及数组的移动；
4. ArrayList不适合做为队列（FIFO），因为会移动数组。但是定长数组很适合做为队列，只需要维持两个头尾指针构成环形数组，如ArrayBlockingQueue。
5. `Arrays.asList()`方法返回的是的`Arrays`内部的ArrayList，用的时候需要注意
6. `subList()`返回内部类，不能序列化，和ArrayList共用同一个数组
7. 迭代删除要用，迭代器的`remove`方法，或者可以用倒序的`for`循环
8. ArrayList重写了序列化、反序列化方法，避免序列化、反序列化全部数组，浪费时间和空间
9. `elementData`不使用`private`修饰，可以简化内部类的访问

