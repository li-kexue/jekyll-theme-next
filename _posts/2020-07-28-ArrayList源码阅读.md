---
layout: post
title: 'ArrayList源码学习'
date: 2020-07-28
author: likexue
photos:
- https://cdn.pixabay.com/photo/2017/06/20/22/14/men-2425121_960_720.jpg
categories: 源码阅读
tags: [源码阅读 ArrayList]
---



对于ArrayList源码，我是初次阅读，可能有很多地方理解不正确，如果有错的话还请大家多多指教。

然后声明一下的我的JDK版本：[openjdk11](http://hg.openjdk.java.net/jdk/jdk11/file/1ddf9a99e4ad/src/java.base/share/classes/java/util/ArrayList.java)。(可以直接打开链接查看，链接的jdk11和我的版本稍有不同，但基本上一致)

### 首先说明一下ArrayList变量的含义

```java
   /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

+ DEFAULT_CAPACITY

> 对于这个值，需要说明一点，当你使用ArrayList<Integer> list = new ArrayList<>();定义一个list时，不是说你不给它赋值，它默认的容量大小就是10了，具体信息会在add函数那里说明。

+ EMPTY_ELEMENTDATA 

> 这个变量主要是当elementData.length为0或者size为0时，为elementData赋一个空数组用的。

+ DEFAULTCAPACITY_EMPTY_ELEMENTDATA

> 对于这个变量，看着好像和EMPTY_ELEMENTDATA没什么区别，但是他俩的作用是不一样的，当我们不指定initialCapacity时，他会用DEFAULTCAPACITY_EMPTY_ELEMENTDATA为elementData进行初始化，之后会通过判断elementData与该变量是不是相等来确定是不是第一次操作。

+ elementData

> 这个变量应该没什么问题，就是用来存储数据的

+ size

> 这个变量用于存储当前ArrayList有多少个元素（注：不是elementData数组的大小）

+ modCount

> 这个变量是来自AbstractList，我感觉很有必要说一下，它是用来记录数组变化的次数（添加删除等操作都算）

### add函数

```java
public void add(int index, E element) {
    this.rangeCheckForAdd(index);
    ++this.modCount;
    int s;
    Object[] elementData;
    if ((s = this.size) == (elementData = this.elementData).length) {
        elementData = this.grow();
    }

    System.arraycopy(elementData, index, elementData, index + 1, s - index);
    elementData[index] = element;
    this.size = s + 1;
}
```

这里**System.arraycopy(elementData, index, elementData, index + 1, s - index)**我觉得与必要说一下，结合System类的注释，大概了解了这个函数的意义。

![]({{site.baseurl}}/assets/images/arraylist/1.gif)

在add函数中，source array和target array其实是同一个数组，这里我为了便于理解，做成了两个。

其中srcPos是拷贝开始的位置，destPos是“粘贴”开始的位置，length是需要拷贝的长度。这里需要注意一下，add函数是确保了数组的容量一定是够的，因为它先进行了扩容的判断，空间不足时，会先进行扩容。

### grow()函数

```java
    private Object[] grow(int minCapacity) {
        //使用Arrays.copyOf将原来的数据拷贝到新的数组中
        return this.elementData = Arrays.copyOf(this.elementData, this.newCapacity(minCapacity));
    }
```

从这可以看出ArrayList扩容还是比较耗时的，如果一开始能够确认数组的大小，那就为他赋一个初始容量，防止它多次扩容而影响性能。

### newCapacity()函数



```java
    private int newCapacity(int minCapacity) {
        int oldCapacity = this.elementData.length;
        //新生成的容量大小是原来的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity <= 0) {
            //如果elementData是第一次扩容，那就在minCapaticy和10中选一个最大值
            if (this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
                return Math.max(10, minCapacity);
            } else if (minCapacity < 0) {
                throw new OutOfMemoryError();
            } else {
                return minCapacity;
            }
        } else {
            //hugeCapacity主要是防止数组大于2147483639
            return newCapacity - 2147483639 <= 0 ? newCapacity : hugeCapacity(minCapacity);
        }
    }
```

对于this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA这句我说是第一次扩容，其实也不完全正确，这里必须是在new一个ArrayList时候没有给它赋初始值，那elementData才会等于DEFAULTCAPACITY_EMPTY_ELEMENTDATA。

上面三段代码就一块分析了，整个添加流程大概如下图：

![]({{site.baseurl}}/assets/images/arraylist/ArrayList.png)

这幅图可能并不是很准确，有些细节我并没有体现，因为怕图越来越臃肿。

### get()函数

```java
    public E get(int index) {
        Objects.checkIndex(index, this.size);
        return this.elementData(index);
    }
```

这个函数很简单，基本没有要介绍的，其中有个Objects.checkIndex(index, this.size)检查下标是否越界。



### hashCode()函数

```java
    public int hashCode() {
        int expectedModCount = this.modCount;
        int hash = this.hashCodeRange(0, this.size);
        //这里不是很确定，应该是防止并发操作导致数组更改,导致求出的hash值与当前值不一致（下面我也写了一个示例）
        this.checkForComodification(expectedModCount);
        return hash;
    }

    int hashCodeRange(int from, int to) {
        Object[] es = this.elementData;
        if (to > es.length) {
            throw new ConcurrentModificationException();
        } else {
            int hashCode = 1;

            for(int i = from; i < to; ++i) {
                Object e = es[i];
                hashCode = 31 * hashCode + (e == null ? 0 : e.hashCode());
            }

            return hashCode;
        }
    }
```

这里解释一下hashCode = 31 * hashCode + (e == null ? 0 : e.hashCode())为什么要乘31.

在Effective Java中作者是这样解释的：

> 乘法部分使得散列值依赖于域的顺序，如果一个类包含多个相似的域，这样的乘法运算就会产生一个更好的散列函数。例如，ArrayList中添加了1,2,3,4这四个元素，另一个ArrayList也添加了这四个元素，但是和第一个顺序不一致，不用乘法的话，那这两个ArrayList的hashCode就是一样的（这里是我自己的理解，下面我也给出了一个例子）。之所以选31是因为他是一个奇素数，如果乘数是偶数，并且乘法溢出的话，信息就会丢失，因为与2相乘等价与移位运算，使用素数的好处并不明显，但习惯上都使用素数来计算散列结果。31有个很好特性，即用移位和减法来代替乘法（$$31 * i == (i << 5) - i$$）



+ checkForComodification的作用

示例1：

![]({{site.baseurl}}/assets/images/arraylist/ArrayList3.jpg)

测试结果：

![]({{site.baseurl}}/assets/images/arraylist/ArrayList4.jpg)

我一开始起了10个线程，发现并没有出现想要的异常，于是直接起1000个线程，不出意外，出现了想要的结果。当然我也不能100%确定就是这个作用，但是依照目前测试结果来看，肯定是有这么个作用的。如果大家有别的见解欢迎指出。

--------------------------------------------------更新--------------------------------------------------------------------------------------------

这里之前只知道在并发情况下会抛异常，不知道具体是什么，最近突然看到一个名词**fail-fast**说的就是上面我提到的情况。

+ 关于hashCode的测试

示例1：

![]({{site.baseurl}}/assets/images/arraylist/ArrayList1.jpg)

测试结果为true。

示例2：

![]({{site.baseurl}}/assets/images/arraylist/ArrayList2.jpg)

结果为false。

可以知道循序不同，hashCode确实是不同的，这也就是乘31所起的作用。

### remove函数

+ remove(int index)

```java
    public E remove(int index) {
         //检查下标是否越界
        Objects.checkIndex(index, this.size);
        Object[] es = this.elementData;
        E oldValue = es[index];
        this.fastRemove(es, index);
        return oldValue;
    }

    private void fastRemove(Object[] es, int i) {
        ++this.modCount;
        int newSize;
        //如果不大于i，说明待删除的是最后一个，直接将其置null
        if ((newSize = this.size - 1) > i) {
            //如果大于，调用本地方法进行删除
            System.arraycopy(es, i + 1, es, i, newSize - i);
        }

        es[this.size = newSize] = null;
    }
```

+ remove(int index)

```java
    public boolean remove(Object o) {
        Object[] es = this.elementData;
        int size = this.size;
        int i = 0;
        //如果待删除的为null，进入if语句
        if (o == null) {
            while(true) {
                //找不到返回false
                if (i >= size) {
                    return false;
                }
				//找到跳出循环
                if (es[i] == null) {
                    break;
                }
                ++i;
            }
            //不为null，进入else（下面同理）
        } else {
            while(true) {
                if (i >= size) {
                    return false;
                }

                if (o.equals(es[i])) {
                    break;
                }

                ++i;
            }
        }
		//找到下标后就可以调用fastRemove了，和上面的remove流程就一样了
        this.fastRemove(es, i);
        return true;
    }
```

