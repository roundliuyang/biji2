# 搞懂 HashSet & LinkedHashSet 源码



经过上两篇的 [HashMap](https://juejin.cn/post/6844903588179755021) 和 [LinkedHashMap](https://juejin.cn/post/6844903590159450120) 源码分析以后，本文将继续分析 JDK 集合之 Set 源码，由于有了之前的 Map 源码分析的铺垫，Set 源码就简单很多了，本文的篇幅也将比之前短很多。查看 Set 源码的构造参数就可以知道，Set 内部其实维护的就是一个 Map，只是单单使用了 Entry 中的 key 。那么本文将不再赘述内部数据结构，而是通过部分的源码，来说明两个 Set 集合与 Map 之间的关系。本文将从以下几部分叙述：

1. **Set 集合概述**
2. **HashSet 源码简单分析**
3. **LinkedHashSet 源码简单分析**
4. **关于面试中的集合问题总结**



## Set 集合概述



由于本篇文章主要叙述 Set 容器以及和 Map 容器之间关系，我们只需要关注上述集合图谱中 Set 部分。可以看出 Set 主要的实现类有 `HashSet` 和 `TreeSet` 以及没有画出的 `LinkedHashSet`。其中 `HashSet` 的实现依赖于 `HashMap`， `TreeSet` 的实现依赖于 `TreeMap`，`LinkedHashSet` 的实现依赖于 `LinkedHashMap`。

从各个实现类的声明也可以看出其继承关系

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
    
    
public class LinkedHashSet<E>
       extends HashSet<E>
       implements Set<E>, Cloneable, java.io.Serializable 
    
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable  
```

在看 Set 的源码之前，我们先概括的说下 Set 集合的特点

HashSet 底层是数组 + 单链表 + 红黑树的数据结构

LinkedHashSet 底层是 数组 + 单链表 + 红黑树 + 双向链表的数据结构

Set 不允许存储重复元素，允许存储 null

HashSet 存储元素是无序且不等于访问顺序

LinkedHashSet 存储元素是无序的，但是由于双向链表的存在，迭代时获取元素的顺序等于元素的添加顺序，注意这里不是访问顺序





## HashSet 的源码分析



HashSet 源码只有短短的 300 行，上文也阐述了实现依赖于 HashMap，这一点充分体现在其构造方法和成员变量上。我们来看下 HashSet 的构造方法和成员变量：

```java
 // HashSet 真实的存储元素结构
 private transient HashMap<E,Object> map;

 // 作为各个存储在 HashMap 元素的键值对中的 Value
 private static final Object PRESENT = new Object();
    
 //空参数构造方法 调用 HashMap 的空构造参数  
 //初始化了 HashMap 中的加载因子 loadFactor = 0.75f
 public HashSet() {
        map = new HashMap<>();
 }
 
 //指定期望容量的构造方法
 public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
 }
 //指定期望容量和加载因子
 public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
 }
 //使用指定的集合填充Set
 public HashSet(Collection<? extends E> c) {
        //调用  new HashMap<>(initialCapacity) 其中初始期望容量为 16 和 c 容量 / 默认 load factor 后 + 1的较大值
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
 }

 // 该方法为 default 访问权限，不允许使用者直接调用，目的是为了初始化 LinkedHashSet 时使用
 HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
 }

```

通过 HashSet 的构造参数我们可以看出每个构造方法，都调用了对应的 HashMap 的构造方法用来初始化成员变量 map ，因此我们可以知道，HashSet 的初始容量也为 `1<<4` 即16，加载因子默认也是 0.75f。

我们都知道 Set 不允许存储重复元素，又由构造参数得出结论底层存储结构为 HashMap,那么这个不可重复的属性必然是有 HashMap 中存储键值对的 Key 来实现了。在分析 HashMap 的时候，提到过 HashMap 通过存储键值对的 Key 的 hash 值（经过扰动函数hash()处理后）来决定键值对在哈希表中的位置，当 Key 的 hash 值相同时，再通过 equals 方法判读是否是替换原来对应 key 的 Value 还是存储新的键值对。那么我们在使用 Set 方法的时候也必须保证，存储元素的 HashCode 方法以及 equals 方法被正确覆写。

HashSet 中的添加元素的方法也很简单，我们来看下实现：

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

可以看出 add 方法调用了 `HashMap` 的 put 方法，构造的键值对的 key 为待添加的元素，而 Value 这时有全局变量 `PRESENT` 来充当，这个`PRESENT`只是一个 Object 对象。

除了 add 方法外 `HashSet` 实现了 Set 接口中的其他方法这些方法有：

```java
public int size() {
        return map.size();
}

public boolean isEmpty() {
   return map.isEmpty();
}

public boolean contains(Object o) {
   return map.containsKey(o);
}

//调用 remove(Object key)  方法去移除对应的键值对
public boolean remove(Object o) {
   return map.remove(o)==PRESENT;
}

public void clear() {
   map.clear();
}

// 返回一个 map.keySet 的 HashIterator 来作为 Set 的迭代器
public Iterator<E> iterator() {
   return map.keySet().iterator();
}
```

关于迭代器我们在讲解 `HashMap` 中的时候没有详细列举，其实 `HashMap` 提供了多种迭代方法，每个方法对应了一种迭代器，这些迭代器包括下述几种，而 `HashSet` 由于只关注 Key 的内容，所以使用 `HashMap` 的内部类 `KeySet` 返回了一个 `KeyIterator` ，这样在调用 next 方法的时候就可以直接获取下个节点的 key 了。

```java
//HashMap 中的迭代器

final class KeyIterator extends HashIterator
   implements Iterator<K> {
   public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator
   implements Iterator<V> {
   public final V next() { return nextNode().value; }
}

final class EntryIterator extends HashIterator
   implements Iterator<Map.Entry<K,V>> {
   public final Map.Entry<K,V> next() { return nextNode(); }
}
```

关于 HashSet 中的源码分析就这些，其实除了一些序列化和克隆的方法以外，我们已经列举了所有的 HashSet 的源码，有没有感觉巨简单，其实下面的 LinkedHashSet 由于继承自 HashSet 使得其代码更加简单只有短短100多行不信继续往下看。



## LinkedHashSet 源码分析



在上述分析 `HashSet` 构造方法的时候，有一个 default 权限的构造方法没有讲，只说了其跟 LinkedHashSet 构造有关系，该构造方法内部调用的是 LinkedHashMap 的构造方法。

LinkedHashMap 较之 HashMap 内部多维护了一个双向链表用来维护元素的添加顺序：

```java
// dummy 参数没有作用这里可以忽略
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
   map = new LinkedHashMap<>(initialCapacity, loadFactor);
}

//调用 LinkedHashMap 的构造方法，该方法初始化了初始起始容量，以及加载因子，
//accessOrder = false 即迭代顺序不等于访问顺序
public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
}
```

LinkedHashSet的构造方法一共有四个，统一调用了父类的 `HashSet(int initialCapacity, float loadFactor, boolean dummy)`构造方法。

```java
//初始化 LinkedHashMap 的初始容量为诶 16 加载因子为 0.75f
public LinkedHashSet() {
   super(16, .75f, true);
}

//初始化 LinkedHashMap 的初始容量为 Math.max(2*c.size(), 11) 加载因子为 0.75f 
public LinkedHashSet(Collection<? extends E> c) {
   super(Math.max(2*c.size(), 11), .75f, true);
   addAll(c);
}

//初始化 LinkedHashMap 的初始容量为参数指定值 加载因子为 0.75f 
public LinkedHashSet(int initialCapacity) {
   super(initialCapacity, .75f, true);
}
 
 //初始化 LinkedHashMap 的初始容量,加载因子为参数指定值 
 public LinkedHashSet(int initialCapacity, float loadFactor) {
   super(initialCapacity, loadFactor, true);
}
```

完了..没错，LinkedHashSet 源码就这几行，所以可以看出其实现依赖于 LinkedHashMap 内部的数据存储结构。



