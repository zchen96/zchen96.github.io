---
layout:     post
title:      LinkedHashMap 源码阅读
subtitle:   jdk、LinkedHashMap、双向链表
date:       2019-01-02
author:     zchen
header-img: img/post-bg-debug.png
catalog: true
tags:
    - jdk
---

如果你对 `HashMap` 的源码有了解的话，只需要一图就能知道 `LinkedHashMap` 的原理了，但是具体的实现细节还是需要去读一下源码。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222182021127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=,size_16,color_FFFFFF,t_70)

### 一、LinkedHashMap 简介

**1.1 继承结构**

从继承结构上来讲 `LinkedHashMap` 继承自 `HashMap`，`LinkedHashMap` 中没有提供任何增删改查的方法，而是直接复用了父类 `HashMap` 中的方法。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181222135743397.png)

**1.2 内部数据结构** 

`LinkedHashMap` 继承自 `HashMap`，`HashMap` 底层存储键值对的数据结构是 `Node` 和 `TreeNode`，而 `LinkedHashMap` 存储键值对的数据结构是 `Entry` 和 `TreeNode`。

``` java
    // Entry 继承自 HashMap 中的 Node 类，用于链表存储键值对
    static class Entry<K,V> extends HashMap.Node<K,V> {
        // 前置与后置节点，用于维护 LinkedHashMap 的键值对顺序
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

`Entry` 用于存储链表中的键值对，用 `before` 与 `next` 指针维护插入键值对的顺序，那树节点是怎么维护插入顺序的呢？在 `LinkedHashMap` 中是没有 `TreeNode` 类的，因为它复用了 `HashMap` 中的 `TreeNode`，下面我们来看一下 `HashMap` 中 `TreeNode` 的定义。

我们只看这一行代码就行了，你会发现 `HashMap` 中 `TreeNode` 类继承了 `LinkedHashMap` 的 `Entry`，这样一来就继承了父类的前置与后置指针，也就能维护 `LinkedHashMap` 的插入顺序了。

``` java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>
```

**1.3 构造函数**

我们只看其中一个构造函数

``` java
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        // 调用 HashMap 的构造函数
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

`LinkedHashMap` 只有这一个构造函数可以初始化 `accessOrder`，这个 `accessOrder` 属性又有什么作用呢，下面来解释一下，看了解释大家可能还会有点迷惑，后面结合代码大家就能理解这个字段的作用了。

``` java
    /*
     * 键值对迭代顺序策略
     * true：access-order     访问顺序，使用迭代器遍历时，get 的元素会被添加到最后
     * false：insertion-order 插入顺序
     */
    final boolean accessOrder;
```

上面讲了那么多都是为相关源码分析作准备，下面我们就一起来看看吧。

### 二、添加键值对

上面我们也说了 `LinkedHashMap` 中没有 `put` 方法，因为都是复用的 `HashMap` 的，如果你想具体了解 `put` 方法，可以查看 [HashMap 扩容机制与线程安全分析](https://zchen96.github.io/2018/12/15/HashMap-%E6%89%A9%E5%AE%B9%E6%9C%BA%E5%88%B6%E4%B8%8E%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E5%88%86%E6%9E%90/) 这篇文章。

想要把键值对放到 `LinkedHashMap` 中去，一定会先封装成 `Entry` 或 `TreeNode` 节点，知道这个原理我们直接来看 `newNode`（以链表节点为例） 方法。

``` java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        // 初始化 LinkedHashMap 键值对 Entry
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        // 添加的键值对都会被添加到 LinkedHashMap 尾部，因此可以形成一个双向链表
        linkNodeLast(p);
        return p;
    }
```

在 `newNode` 方法初始化 `Entry` 节点时调用了一个 `linkNodeLast` 方法，我想看到这里大家应该这道其中的原理了。

``` java
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        // 获取并记录 tail 节点
        LinkedHashMap.Entry<K,V> last = tail;
        // 重置 tail
        tail = p;
        if (last == null)
            head = p;
        else {
            // 将节点连接起来，构成双向链表
            p.before = last;
            last.after = p;
        }
    }
```
### 三、获取键值对

`LinkedHashMap` 的 `get` 方法调用了 `HashMap` 的 `getNode` 方法，获取到对应的节点后，返回对应的 value。

``` java
    public V get(Object key) {
        Node<K,V> e;
        // 调用 hashMap 中的 getNode() 方法，根据 key 的哈希值找到对应的桶位置，判断节点后（链表、头结点、树节点）进行返回
        if ((e = getNode(hash(key), key)) == null)
            return null;
        // 如果 accessOrder 为 true，获取元素后把当前键值对调整到尾部
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

下面我们来看一下 `getNode` 方法都做了什么。这个方法做的事情很简答， 根据 key 计算出对应的哈希值，根据哈希值计算出对应的桶位置，然后遍历节点进行查找。

``` java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 进行判断并通过 tab[(n - 1) & hash] 计算当前 key 在哈希表中的位置
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 如果是当前桶位置上的头节点直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 如果头节点不是要找的节点，判断是树节点还是链表节点后继续查找
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 链表、从头节点开始遍历查找
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
上面的 `get` 方法中还有一个需要注意的地方，如果 `accessOrdera` 为 `true` 则调用 `fterNodeAccess(e);` 方法，这个方法又是干嘛用的呢，在最开始的时候我们说过，`LinkedHashMap` 支持两种迭代策略，分别是访问顺序（默认）与插入顺序，这个 `fterNodeAccess(e);` 方法就是用于支持访问顺序的。

如果 `accessOrdera` 为 `true`，那么在调用 `get` 方法时会将该键值对置为双向链表的尾节点。

``` java
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        // 判断迭代策略，并且当前节点不是尾节点
        if (accessOrder && (last = tail) != e) {
            // 记录当前节点，并获取前后节点
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            // 把当前节点的 after 节点置 null
            p.after = null;
            // 如果当前节点是头节点，把后一个节点置为头节点
            if (b == null)
                head = a;
            // 把当前节点的前后节点相连
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            // 把当前节点置为尾节点并记录
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

### 四、HashMap 与 LinkedHashMap 对比

除了双向链表外，`HashMap` 与 `LinkedHashMap` 还有什么区别吗？

**4.1 containsValue 方法对比**

`HashMap` 中代码如下，遍历哈希表中的所有桶，然后遍历桶位置上的所有节点。

``` java
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            // 遍历哈希表
            for (int i = 0; i < tab.length; ++i) {
                // 遍历当前桶上的所有节点
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

`LinkedHashMap` 中代码如下，从头节点遍历，一直遍历到尾节点。

``` java
    public boolean containsValue(Object value) {
        /**
         * 遍历双向链表，判断 value，与 HashMap 完全不同
         */
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```

**4.2 nextNode 方法对比**

`HashMap` 中的 `nextNode()` 方法，遍历哈希表数组，接着遍历桶位置上的所有节点。

``` java
     final Node<K,V> nextNode() {
          // 记录哈希表数组
         Node<K,V>[] t;
         Node<K,V> e = next;
         if (modCount != expectedModCount)
             throw new ConcurrentModificationException();
         if (e == null)
              throw new NoSuchElementException();
          // 过滤掉没有键值对的桶位置
         if ((next = (current = e).next) == null && (t = table) != null) {
              do {} while (index < t.length && (next = t[index++]) == null);
         }
          // 下一个有键值对的桶（单个节点、树节点或链表）
         return e;
     }
```
 
`LinkedHashMap` 中的 `nextNode()` 方法，根据双向链表的顺序迭代键值对。

``` java
    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        // next 为双向链表的下一个节点
        next = e.after;
        return e;
    }
```

jdk1.8 源码阅读：[https://github.com/zchen96/jdk1.8-source-code-read](https://github.com/zchen96/jdk1.8-source-code-read)

### 参考资料

[搞懂 Java LinkedHashMap 源码](https://juejin.im/post/5ace2bde6fb9a028e25deca8)