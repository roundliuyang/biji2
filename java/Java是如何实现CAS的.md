# Java是如何实现CAS的



## 前言



在java面试中，经常会被问到`i++是不是一个原子性操作？`，或者`以下代码的执行结果是什么?`

```java
private volatile static int i = 0;
  public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
      // 1000次循环i++s
      for (int x = 0; x < 1000; x++) {
        i++;
      }
    });
    Thread t2 = new Thread(() -> {
      // 1000次循环i++
      for (int x = 0; x < 1000; x++) {
        i++;
      }
    });
    // 启动线程t1，t2
    t1.start();
    t2.start();
    // 等待t1，t2执行结束
    t1.join();
    t2.join();
    // 打印i的数值
    System.out.println(i);
  }
```

很显然，在大多数情况下，i的值基本都小于2000，且是一个不固定的值。为了解决这个问题，大部分程序员都会提出使用`AtomicInteger`来解决：

```java
// 使用AtomicInteger包装
private static AtomicInteger i = new AtomicInteger(0);
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
      for (int x = 0; x < 1000; x++) {
        // 自增
        i.getAndIncrement();
      }
    });
    Thread t2 = new Thread(() -> {
      for (int x = 0; x < 1000; x++) {
        // 自增
        i.getAndIncrement();
      }
    });
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    // 打印i的值
    System.out.println(i.get()); 
}
```

通过上述代码修改后，我们得到的i的值一定是自增2000次后的结果。那么为什么`AtomicInteger`包装一下后就能满足原子性呢？



## CAS的执行流程



为了解答我们的疑问，我们需要简单地看一下`AtomicInteger`的源码：

```java
public final int getAndIncrement() {
    return U.getAndAddInt(this, VALUE, 1);
}
```

好吧，就一行代码，确实看不出什么原理，只能再进一步深入：

```java
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

我们可以发现`getIntVolatile()`是一个`native`方法，这个方法是由底层的`C++`语言实现的；但是通过两个参数的含义基本可以猜测到：`getIntVolatile()将会从对象o的相对偏移量offset取值`，那么这个取出来的值就是内存中的数值v；

```java
@HotSpotIntrinsicCandidate
public final boolean weakCompareAndSetInt(Object o, long offset, int expected, int x) {
    return compareAndSetInt(o, offset, expected, x);
}
```

另外，我们发现`compareAndSetInt()`也是`native`方法，不过根据源码注释我们依然还是可以看明白这个方法的意思：`对比对象o偏移量offset位置的值，看看是不是和expected一致，如果一致的话，就把偏移量offset位置的值修改为x`，并且`compareAndSetInt()`这个操作是原子性的。

那么根据以上两段代码的解析，我们就可以大致分析出CAS的执行流程了：

> 1.先通过对象的偏移量找到目标值；
>
> 2.判断找到的目标值是否和内存中偏移量位置的值一致（为避免在执行间隙有其他线程执行导致内存值产生变化，所以要时刻保持与内存值比较），如果一致，说明这个时间差内没有其他线程修改内存值，那么就可以放心地把内存值改成我们的目标值`x`；
>
> 3.如果修改失败，那么说明内存值产生了变化，那么重新执行步骤1；

以上就是我们根据源码分析总结出来的CAS的执行流程，那么我们还有一个疑问，对象的偏移量是怎么计算出来的呢？





## CAS的实现原理（基石Unsafe）



在`AtomicInteger`类中，我们会看到这样三段代码：

```java
// Unsafe对象，只能在Boostrap ClassLoader下才能被创建
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
// 字段value在AtomicInteger对象中的偏移量
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
// 被包装的目标字段
private volatile int value;
```

在java中，每一个字段在内存中都会占据一定大小的位置，int类型的也是一样。`Unsafe`对象就是根据这个原理，通过计算这个字段在类中的相对偏移量，那么在对象被创建出来后，就能够根据这个早就计算好的相对偏移量来直接获取指定字段在内存中的值了。

通过以上分析，我们很自然地就知道了，原来CAS的实现是基于Unsafe能够获取字段在对象中的相对偏移量，并通过不断地自旋操作比较内存值是否发生变化来决定是否更改内存中的目标值。





在`ConcurrentHashMap`类中，我们会看到这样的代码：

```java
/**
 * 通过CAS算法添加tab数组中指定位置的数据
 * @param tab   tab数组
 * @param i     hash值
 * @param c     预期值
 * @param v     新值
 * @return     是否添加成功
 */
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

```
int  i = 6;
long l = (long) i << 2;
res：l =24
```



## Unsafe类

```java
/**
 * 更新变量值为x，如果当前值为expected
 * o：对象 offset：偏移量 expected：期望值 x：新值
 */
public final native boolean compareAndSwapObject(Object o, long offset,
                                                 Object expected,
                                                 Object x);
```

