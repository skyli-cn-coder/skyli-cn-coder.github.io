---
title: CopyOnWriteArrayList 浅析
date: 2019-09-13 22:12:45
categories: JSE
tags: [JUC]
---

### 概述

CopyOnWriteArrayList（COWAL, 文中将引用该命名） 和 ArrayList 在功能概念上是相同，都是可动态扩容的线性数组。但是两种数据结构的应用场景不同：

* COWAL: 多线程下访问列表是线程安全的，该数据结构通常应用在读操作场景多于写操作场景，具体原因稍后源码中分析指出。
* AarrayList: 多线程下访问列表非线程安全，针对读写操作无特殊要求。

### 源码解析

首先从名称上可以看出一些规律， ```CopyOnWrite``` 说明当向list中写入数据时可能进行了 copy 操作。至于 copy 了什么？为什么需要 copy?。我们接着往下看：
1、首先看下底层数据结构：

``` java
 /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
    // 该数据结构的底层同ArrayList相同都是一个数组
```

说明 COWAL 同 ArrayList 一样，底层用 Object 数组存放数据。

2、然后我们再来看下 COWAL 的相关 API

* 无参构造器：无参构造器会初始化一个长度为 0 的 Object 数组。

``` java
/**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]); // 这里通过 setArray 方法将新创建的数组引用复制给底层数组。
    }
```

* 有参构造器：有参构造器参数为 Collection<? extends E> c

``` java
 public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }
```

从源码中我们可以看到每次构造 COWAL 对象时都会声明了一个名称为 elements 的数组，然后将另一个数组的引用赋值给 elements; 这里的另一个数组可能是 c 强转后的数组，也有可能是 通过 Arrays.copyOf 方法新创建的。

* 接下来我们看下COW 的相关 API

**1、首先传统容器存在的问题：**

我们知道在 ArrayList 中多线程操作数据非线程安全的，JDK 有对应的线程安全的相关容器 Vector 其实现原理是将具体的操作方法前加 Syncronized 关键字来保证线程安全性。

但是 Vector 在多线程环境下使用不当依然会出现线程安全问题：

``` java
 public static void main(String[] args) {

        Vector<String> vector = new Vector<>();

        vector.add("hello");
        vector.add("hello");
        vector.add("hello");
        vector.add("hello");
        vector.add("hello");
        vector.add("hello");

        for (int i = 0; i < vector.size(); i++) {
            new Thread(vector::clear).start();
            System.out.println(vector.get(i));
        }
    }
```

运行结果：

``` java
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 0
	at java.util.Vector.get(Vector.java:748)
	at com.skyli.Simulator.main(ConfigManager.java:93)

Process finished with exit code 1
```

从上面的运行效果可以看到，及时 Vector 中的所以操作方法都添加了 synchronized 关键词来同步方法的执行，但是在同步代码执行顺序不合理时是会导致代码运行异常的。错误原因： 当循环执行 vector.size() 方法后，如果其他线程对 vector 有修改的话，则会出现数组越界异常。解决的办法是在循环前强制添加 synchronized 关键词， 即：

``` java
 synchronized(vector) {
        for (int i = 0; i < vector.size(); i++) {
            new Thread(vector::clear).start();
            System.out.println(vector.get(i));
        }
  }
```

相信有的童鞋可能已经发现不妥了，简单的遍历下容器就要加一个锁，那执行效率岂不是很慢。。。

那么本文的 COWAL 就是解决这样场景下的 problem 的：

**2、我们先来看下 COWAL 的 API**

``` java
 public boolean add(E e) {
        final ReentrantLock lock = this.lock; // 使用可重入锁对添加操作上锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1); // 将 COWAL 对象的底层数组复制到一个新数组中 
            newElements[len] = e; // 添加动作在新数组上执行
            setArray(newElements); // 将新数组的引用赋值给 COWAL 的底层数组对象
            return true;
        } finally {
            lock.unlock();// 操作执行完成后释放锁
        }
    }
```
从向容器中添加新元素的源码中我们可以看到，COWAL 会新创建一个新数组并把老数组中的数据 copy 过去，将新数组作为操作后的底层数据结构。

这种 copy（生成副本）的操作只有在试图更改 COWAL 的结构时（底层数据结构的元素数量发生变化）才会触发。

同理看下 remove 方法：

``` java
 public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray(); // 老数组
            int len = elements.length;
            E oldValue = get(elements, index); // 从老数组中取得要删除的元素
            int numMoved = len - index - 1; // 判断删除操作导致需要移动的元素数量
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));// 不需要移动时，则说明删除的是数组的最后一个元素，则直接将剩余元素复制到新创建的数组中
            else {
                Object[] newElements = new Object[len - 1];// 创建容量为老数组容量 - 1 的新数组
                System.arraycopy(elements, 0, newElements, 0, index);// 将老数组中要删除元素位置前的所有元素复制到新数组中
                System.arraycopy(elements, index + 1, newElements, index,// 将老数组中要删除元素位置之后的元素复制到新数组中
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();// 操作完成释放锁
        }
    }
```

所以由上面的源码可以看到，凡是针对容器进行结构性变化的操作时，COWAL 大致做如下操作：

1. 给操作方法代码块上锁
2. 创建一个新数组
3. 在新数组上执行操作逻辑

所以针对 COWAL 中的所以结构性操作再多线程环境下是同步的。但是为什么在遍历 COWAL 时不显示加锁也不会出现线程安全的问题呢：

下面我们来简单剖析下：

首先，看下 COWAL 的迭代器吧：

``` java
public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0); // new 一个 COW 的迭代器
    }

...

 static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot; // 一个快照数组
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }
        ...
```

从上面的源码可以看出迭代器中的所有操作都是在成员属性 snapshot 上进行的。也就是在 COWAL 的老数组上进行操作。所以在 COWAL 没有发生结构性变化时，所有的迭代器指向的是同一个数据源，当 COWAL 发生结构行变化时，由于结构性变化是在复制的新数组上进行，所以不影响老数组的一些遍历操作。


这里需要注意下，由于所有的迭代器指向的是同一个数据源，所以为了保证线程安全 COWAL 是不允许在 迭代器中执行 set 与 remove 操作的：

``` java
 /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code set}
         *         is not supported by this iterator.
         */
        public void set(E e) {
            throw new UnsupportedOperationException();
        }
```

经过上面的分析，童鞋们是否能总结出 COWAL 的优缺点呢？

首先说下优点吧：

1. COWAL 在不显示加锁的情况下是线程安全的，在读多写少的场景中，其效率远远高于 Vector 或者 Collection.synchronizend(List)
2. 使用简单、高效（这个最重要）

任何事物往往具有两面性，有优点就会有缺点：

1. COWAL 是不适合在写多读少的场景中使用的，因为 COWAL 在发生结构性变化的时候会新创建一个数组并把老数组的数据 copy 到新数组中，这种操作会很占用内存，很容易触发OOM。
2. COWAL 保证的不是实时一致性，其保证的是最终一致性。先遍历操作不会读到后插入的数据。

针对上面的 COWAL 的缺点第二项简单已代码进行演示：

``` java
 public static void main(String[] args) {
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<>(new Integer[]{1, 3, 5, 8});

        Iterator<Integer> iterator = list.iterator();

        list.add(10);

        while (iterator.hasNext()) {
            System.out.print(iterator.next() + ",");
        }
    }
    
 运行结果：
 1,3,5,8,
 Process finished with exit code 0
```

从上面的示例中可以看到， 当迭代器创建成功后， list 的任何结构性变化是不会影响到当前迭代器的遍历操作的，这也是遍历结果与实际结果不一致的体现，但是 list 是能够保证最终一致性的，当再次创建一个新的迭代器时便能够遍历到结构性变化导致的最终结果。

好了，CopyOnWriteArrayList 的简单剖析就到这里了，最后抛一个小小的问题：
大家有没有注意到 COWAL 的底层数组使用了 volatile 修饰。那为什么要用这个修饰符呢，如果不用会有什么问题吗？欢迎一起来讨论。

```java 
/** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

