---
title: LinkedList源码分析
date: 2020-08-01 13:59:58
categories:
- Java
- 集合框架
tags:
- Java
- 数据结构
---



## 简介

`LinkedList`底层采用双向链表结构实现，所以在存储元素，并不需要扩容机制，但是需要额外的空间存储指针，头插和尾插的时间复杂度为`O(1)`，指定位置插入的时间复杂度为`O(n)`，`LinkedList`是非线程安全的集合。

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
```



`LinkedList`继承了`AbstractSequentialList`。该类提供了一套基于顺序访问的接口。

实现了`List`接口和`Deque`接口，使得`LinkedList`同时具备了`List`和双端队列的特性。

`LinkedList`实现了`Serializable`接口，表明`ArrayList`支持序列化。

`LinkedList`实现了`Cloneable`接口，能被克隆。



## 数据结构

Node节点包括数据，前驱节点和后继节点。

```java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```



## 成员变量

分别是链表长度，头节点，尾节点。

```java
transient int size = 0;

transient Node<E> first;

transient Node<E> last;
```





## 构造方法



构造方法有两种，注释如下：

```java

	//空构造方法
	public LinkedList() {
    	}

    //传入集合，调用addAll进行添加
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

```





## API



### linkFirst

将元素添加到头部。

```java
private void linkFirst(E e) {
        final Node<E> f = first;
    	//新节点的前驱节点为null，后继节点为原来的头节点
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
    	//如果头节点为空
        if (f == null)
            //新插入的既是头节点，也是尾节点。
            last = newNode;
        else
            //将原来头节点的前驱指针指向新节点
            f.prev = newNode;
        size++;
        modCount++;
    }
```



### linkLast

将元素添加到尾部。

```java
void linkLast(E e) {
        final Node<E> l = last;
    	//新节点的后继节点为null，前驱节点为原来的尾节点
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
    	//如果尾节点为空,直接将新节点设置为头节点
        if (l == null)
            first = newNode;
        else
            //否则将原来的尾节点后继指向新节点
            l.next = newNode;
        size++;
        modCount++;
    }
```



### linkBefore

在一个非空节点前插入元素。

```java
void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
    	//插入节点的前驱为succ的前驱，后继节点为succ
        final Node<E> newNode = new Node<>(pred, e, succ);
    	//设置succ的前驱指针
        succ.prev = newNode;
    	//如果succ的前驱节点为空
        if (pred == null)
            //新插入的节点为头节点
            first = newNode;
        else
            //否则succ的前驱节点的后继指针指向新节点
            pred.next = newNode;
        size++;
        modCount++;
    }
```



### unlinkFirst

移除头节点。

```java
private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
    	//获取头节点的下一个元素
        final Node<E> next = f.next;
    	//方便GC
        f.item = null;
        f.next = null;
    	//first指针指向next节点
        first = next;
    	//如果链表只有一个节点
        if (next == null)
            //删除后为空，将尾指针置空
            last = null;
        else
            //否则将next的前置置为空
            next.prev = null;
    	//设置size和modCount
        size--;
        modCount++;
        return element;
    }
```





### unlinkLast

移除尾节点。

```java
private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
    	//方便GC
        l.item = null;
        l.prev = null;
    	//last指针指向原来尾节点的前一个
        last = prev;
    	//如果前一个为空
        if (prev == null)
            //说明现在没有节点，头节点置空
            first = null;
        else
            //否则将尾节点的next置为空
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```





###    unlink

移除一个非空节点。

```java
E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
		//如果x是头节点，将头指针指向x的next
        if (prev == null) {
            first = next;
        } else {
            //不是的话，将x的前驱指针指向它的后继节点
            //并将x的前去指针置空
            prev.next = next;
            x.prev = null;
        }
		//如果后继节点为空，说明删除的节点是尾节点
        if (next == null) {
            //last指向前一个
            last = prev;
        } else {
            //处理next的前驱指针
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```





### node

根据下标返回节点。

```java
Node<E> node(int index) {
        //如果下标小于size/2，从头节点开始遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            //从尾部开始遍历
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```



### indexOf

返回元素第一次出现的下标。

```java
public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            //o为null时，找到第一个null节点
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            //否则从头开始找与o匹配的
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```





### lastIndexOf

找到最后一个匹配的下标。

```java
public int lastIndexOf(Object o) {
        int index = size;
    	//从尾部开始找
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }
```





### add

add方法调用`linkLast`，将元素添加到链表。

```java
public boolean add(E e) {
        linkLast(e);
        return true;
    }


void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```





### addAll

将传入的集合从指定下标开始插入。

```java
public boolean addAll(int index, Collection<? extends E> c) {
    	//检查index
        checkPositionIndex(index);
		//存到数组中
        Object[] a = c.toArray();
        int numNew = a.length;
    	//检验是否为空数组
        if (numNew == 0)
            return false;
		
        Node<E> pred, succ;
    	//如果插入位置为尾部
        if (index == size) {
            //succ指向空 pred指向尾节点
            succ = null;
            pred = last;
        } else {
            //不为空则通过node方法找到指定下标的节点
            succ = node(index);
            pred = succ.prev;
        }
		//遍历插入数据
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            //如果插入位置在链表头部
            if (pred == null)
                //头指针设置新节点
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }
		//如果从尾部开始插入
        if (succ == null) {
            //重置尾节点
            last = pred;
        } else {
            //否则将插入的链表与原来的链表连接起来
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
```



### peek

peek方法返回链表的头节点。

```java
public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

```



### poll

poll方法返回头节点，并将头节点删除。

```java
public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }
```



## 总结

`LinkedList`通过实现`List `接口与`Deque`接口，底层数据结构采用了双向链表，同时具备了`List`，`Queue`和`Stack`的特性，本文对其该集合的基本的API实现做了简要分析。关于迭代器并未阐述，其机制与`ArrayList`类似，通过`fail-fast`检测在迭代时结构是否发生改变。

与`ArrayList`相比，`LinkedList`的查找性能差于`ArrayList`，但是其插入和删除性能优于`ArrayList`，在日常开发中，可以根据业务场景选择合适的集合。

