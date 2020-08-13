---
title: HashMap源码分析
date: 2020-08-02 17:15:18
categories:
- Java
- 集合框架
tags:
- Java
- 数据结构
---



## 概述

​		`HashMap`是`Map`接口基于哈希的实现，在查询性能上，时间复杂度接近`O(1)`。在`JDK 7`中采用了数组+链表的数据结构，在`JDK 8`后，底层数据结构转变为`数组+链表+红黑树`，也就是当出现Hash冲突时，链表长度过长，转为红黑树存储节点提高搜索效率。`HashMap`为非线程安全的类，多线程处理时会出现问题。



```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

`HashMap` 继承了 `AbstractMap` ，该类提供了Map接口的抽象实现，并提供了一些方法的基本实现。实现了 `Map`、`Cloneable`和 `Serializable`接口。



## 成员变量

- loadFactor：该变量控制table数组存放数据的疏密程度，越趋向1时，数组中存放的数据越多越密。链表的长度会增加，因此会导致查找效率变低。该值越小，则数组中存放的数据越少，越稀疏，则会导致空间利用率下降。默认值0.75是较好的默认值。
- threshold：当前 `HashMap`所能容纳键值对数量的最大值，超过这个值，则需扩容。**threshold = capacity \* loadFactor**
  - 默认容量为16，默认负载因子为0.75，当存放的数据达到16 * 0.75时，需要进行扩容。

```java
	//存储元素的数组，大小为2的幂次
	transient Node<K,V>[] table;

	//存放具体元素的集
    transient Set<Map.Entry<K,V>> entrySet;

	//已经存放了的数组大小
    transient int size;

	//结构修改的计数器
    transient int modCount;

	//临界值，实际大小超过该值，则进行扩容
    int threshold;

	//负载因子
    final float loadFactor;

	//默认初始容量：16
	static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

	//最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //默认的负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //链表节点数大于该阈值，转为红黑树存储
    static final int TREEIFY_THRESHOLD = 8;

    //红黑树节点数小于该值，转为链表存储
    static final int UNTREEIFY_THRESHOLD = 6;
	
	//树化时，检查table数组长度是否大于该值，小于则扩容
    static final int MIN_TREEIFY_CAPACITY = 64;
```



## 数据结构

链表状态下的节点，继承自`Map.Entry<K,V>`。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    //构造方法
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    //Node的HashCode返回键值的HashCode异或值
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
	//设置新值，返回旧值
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
	//equals
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}

//返回key的HashCode值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

//如果对象x的类是C，如果C实现了Comparable<C>接口，那么返回C，否则返回null
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) {
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                     Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}
```



红黑树状态下的节点，继承自`LinkedHashMap.Entry<K,V>`，而该类继承自`HashMap.Node<K,V>`。

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        //父结点
    	TreeNode<K,V> parent;  // red-black tree links
    	//左孩子节点
        TreeNode<K,V> left;
    	//右孩子节点
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
    	//颜色
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

```



## 构造方法

核心的构造方法是第一个，通过调用`tableSizeFor`为`threshold`（扩容临界值）赋值。


```java
	public HashMap(int initialCapacity, float loadFactor) {
        //边界检测
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
        //边界检测
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

	//调用前一个构造方法
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

   //未传参数时,负载因子设置为默认值
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }

    //传递Map的时候，调用putMapEntries进行批量添加
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```



## API


### tableSizeFor

该方法在初始化时，对`threshold`赋值，通过位运算找到大于或等于cap的最小的2的幂次方。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



### hash

该方法在插入时，计算元素的hash值时调用。`hash`值为传入key的`hashCode`与其右移16位的异或值。这被称为**扰动函数**。

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`hashCode`方法，是key的类自带的Hash方法，返回一个int的Hash值，理论上，这个值应当均匀得分布在int的范围内，但是`HashMap`初始化大小为16，如果让hash映射到16个桶中，通过取模实现十分简单。但是直接取模会造成较多的哈希碰撞。通过扰动函数，增加了随机性，减少了碰撞的几率。



那么如何通过hash值，获取对应的数组下标呢？在`putVal`中，获取下标的方法如下：`(n - 1) & hash`，n是数组大小，上文中，`threshold`为2<sup>n</sup>，那么`n - 1`<sub>2</sub> = 00....1111，那么通过`&`运算，会保留下hash的低位。为何用`&`替代取模运算，主要是位运算的效率远远高于取模运算。这也证明了为何`threshold`必须要是2的幂次方，通过控制该值，从而达到提高hash映射效率的目的。

```java
if ((p = tab[i = (n - 1) & hash]) == null)
```



### get

主要逻辑：

1. 根据hash找到指定位置的节点
2. 判断第一个节点的key是否符合要求，符合要求直接返回第一个节点，否则继续查找。
3. 如果是红黑树结构，通过红黑树查找节点数据并返回
4. 如果是链表结构，遍历节点查询并返回
5. 如果没有找到，返回null

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}


final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //第一个节点就命中
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //判断是否为红黑树节点
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //如果是链表，就遍历链表找到相应的数据
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



### put

主要逻辑：

1. 桶数组table为空，通过`resize()`进行扩容。
2. 查找要插入的键值对是否已经存在，存在的话，用新值替换旧值。
3. 如果不存在，将键值对插入链表尾部，并根据长度判断是否将链表转换成红黑树。
4. 判断键值对数量是否大于阈值，大于的话进行扩容操作。

需要注意的是，`treeifyBin`方法在进行树化前，进行了检查。如果小于`MIN_TREEIFY_CAPACITY`，则进行扩容，不进行树化。

```java
if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
```



```java
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            //如果未初始化，进行数组初始化，赋予初始容量
            n = (tab = resize()).length;
    	//通过hash找到下标，如果该位置为空
        if ((p = tab[i = (n - 1) & hash]) == null)
            //直接将数据存储进去
            tab[i] = newNode(hash, key, value, null);
        else {
            //发生hash碰撞
            Node<K,V> e; K k;
            //如果插入的key和当前key相同，将e指向当前键值对
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果当前节点类型是红黑树节点，使用红黑树进行插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //链表的情况
                for (int binCount = 0; ; ++binCount) {
                    //将新节点放到链表的末尾
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度达到红黑树化的阈值，将链表转化成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果该key已经存在于链表中，覆盖
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //e不为空说明值已经插入成功
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent控制是否替换原来的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
    	//扩容检测
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```





### remove

主要逻辑：

1. 判断第一个节点是否是需删除节点，如果是，将节点存储下来。
2. 如果节点是红黑树节点，通过调用`getTreeNode`找到需删除节点，存储下来。
3. 如果是链表，遍历获取到需删除节点。
4. 删除节点，并进行修复工作

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}


final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //如果键与第一个节点相等，则该节点是需删除节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            //如果是红黑树节点，调用红黑树的查找方法找到需删除节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    //遍历链表，找到需删除节点
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //删除节点，并修复链表或红黑树
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```



### resize

`HashMap`的table数组长度为2的幂，阈值大小 = `capacity * load factor`，`Node`数量超过阈值，进行扩容。

主要逻辑：

1. 计算新的桶数组容量`newCap`和新阈值`newThr`。`newCap`为原来的两倍，`newThr`为原来的两倍。
2. 根据`newCap`创建新的桶数组，初始化新的桶数组。
3. 将键值对节点重新映射到新的桶数组中，如果是红黑树节点，则需要拆分红黑树，如果是普通节点，则节点按照顺序进行分组。

需要注意的是：`resize`十分消耗性能，日常开发需要尽量避免。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //如果table不为空，说明已经被初始化
    if (oldCap > 0) {
        //如果table的容量超过最大值，不进行扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //将容量变为原来的两倍，阈值变为原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //newCap = threshold
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        //阈值为默认容量与默认负载因子乘积
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //新阈值为0时，按照默认公式重新算
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //赋值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //创建新的桶数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //如果旧的桶数组不为空，就遍历桶数组，并将键值对映射到新的桶数组
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    //如果是红黑树节点，进行拆分
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 链表节点情况
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        //遍历链表并进行按序分组
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //将分组后的链表映射到新的桶数组中
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
```



## 红黑树退化

在`TreeNode.split`方法和中，有一段代码对是否需要进行退化进行了判断。如果树节点个数小于`6`，则会退化为链表，至于该阈值与树化阈值`（UNTREEIFY_THRESHOLD 与 TREEIFY_THRESHOLD）`不等的原因，主要为了避免桶数组中的某个节点在该值附近震荡，从而导致频繁的树化和链表化。

```java
if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
```



此外，在`TreeNode.removeTreeNode`中，删除红黑树节点之前，如果满足以下条件，也会进行链表化再进行删除：

- 树的左子树为空
- 树的右子树为空
- 树的左孙子节点为空

```java
if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // too small
                return;
}
```



## 小结

本文对 JDK 8 中的 `HashMap` 的源代码进行了简要分析，主要为增删改查接口的内部实现机制以及扩容原理。

HashMap内部基于数组实现的，数组每个元素称为一个桶(bucket)，当存储的键值对数量超过阈值时，还会进行扩容操作，HashMap中的键值对会重新Hash到新位置。当桶中节点数超过阈值，则会进行树化，如果删除导致低于阈值，则会进行链表化。
