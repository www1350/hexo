---
title: 'HashMap源码(jdk1.8) '
abbrlink: afadb629
date: 2018-04-03 22:58:47
tags:
categories:
---

- HashMap的底层主要是基于数组和链表来实现的，它之所以有相当快的查询速度主要是因为它是通过计算散列码来决定存储的位置。HashMap中主要是通过key的hashCode来计算hash值的，只要hashCode相同，计算出来的hash值就一样。如果存储的对象对多了，就有可能不同的对象所算出来的hash值是相同的，这就出现了所谓的hash冲突。学过数据结构的同学都知道，解决hash冲突的方法有很多，HashMap底层是通过链表来解决hash冲突的。
- JDK1.6中HashMap采用的是位桶+链表的方式，即我们常说的散列链表的方式，而JDK1.8中采用的是位桶+链表/红黑树的方式，也是非线程安全的。当某个位桶的链表的长度达到某个阀值的时候，这个链表就将转换成红黑树。也就是说原来如果hash不理想，所有都落入同一个桶就变成的单链表，现在只要大于8个在同一个桶里面，就会转化为红黑树，提升效率。
- 原来jdk1.7中resize是通过链表头插入的，jdk1.8是通过链表尾插入。可能有人会觉得1.7这样会更快，但是这样容易产生hash碰撞。如果它知道我们用的是哈希算法，它可能会发送大量的请求，导致产生严重的哈希碰撞。然后不停的访问这些 key就能显著的影响服务器的性能，这样就形成了一次拒绝服务攻击（DoS）

<!-- more -->

![image](https://cloud.githubusercontent.com/assets/7789698/18426598/702f1838-78f5-11e6-80ff-47cba88423ed.png)

```
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{
//默认初始化容量，HashMap容量必须是2的幂次方
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//最大容量不得超过1<<30
    static final int MAXIMUM_CAPACITY = 1 << 30;

//默认装载因子，0.75是权衡空间和时间开销之后的综合考虑
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

//超过这个阈值将使用红黑树组织桶中的结点，而不是链表
    static final int TREEIFY_THRESHOLD = 8;

    static final int UNTREEIFY_THRESHOLD = 6;

//只有表的大小超过这个阈值，桶才可以被转换成树而不是链表（为超过这个值时，应该使用resize）
//这个值是TREEIFY_THRESHOLD的4倍，以便resizing和treeification之间产生冲突
    static final int MIN_TREEIFY_CAPACITY = 64;


    transient Node<K,V>[] table;//存储元素的实体数组

    transient Set<Map.Entry<K,V>> entrySet;

    transient int size;

    transient int modCount;//被修改的次数

    int threshold;//临界值   当实际大小超过临界值时，会进行扩容threshold = 加载因子*容量

    final float loadFactor;//加载因子

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
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }


    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    static final int tableSizeFor(int cap) {
        int n = cap - 1; //减1是为了排除“100000”这种情况
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

}
```

Node类似一个单链表

```
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

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
```

```
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
//为空则resize
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
//如果hash不到（桶是空的）就建立一个新结点放入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
//桶中有东西了
        else {
            Node<K,V> e; K k;
　//如果散列到的结点散列值一样且key一样，将桶中元素记录下来
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
//红黑树结点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
//链表,一个个取，取到末尾,如果超过链表定义最大界限，转成红黑树
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
//树化
                            treeifyBin(tab, hash);
                        break;
                    }
//散列值一样的,直接跳出
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
//如果map中存在一样的元素
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
// onlyIfAbsent为false或者旧值为null 
//用在如，(onlyIfAbsent 为true，如果存在则不会替换)
//    @Override
 //   public V putIfAbsent(K key, V value) {
//        return putVal(hash(key), key, value, true, true);
 //   }
//替换成新值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
//插入后回调
        afterNodeInsertion(evict);
        return null;
    }

```



resize
![image](https://cloud.githubusercontent.com/assets/7789698/18426619/8fa7f77a-78f5-11e6-91cd-bc0dca0a3bbd.png)

```
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
//原数组不为空
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {//如果超过最大了2^32
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 两倍
        }
 // 桶数组为空，第一次分配，结合不同构造器的情况细节稍有不同：
        else if (oldThr > 0)
            newCap = oldThr;
        else {               //为空，且不是第一次分配了
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
//无元素
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;


        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
//如果只有一个元素，rehash并放入桶中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else {
/*桶中存在一个链表，需要将链表重新整理到新表当中，因为newCap是oldCap的两倍所以原节点的索引
值要么和原来一样，要么就是原(索引+oldCap)和JDK 1.7中实现不同这里不存在rehash，直接使用原hash
值JDK 1.7中resize过程是在链表头插入，这里是在链表尾插入*/

                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
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

LinkedHashMap里面的Entry很神奇的。

```
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```


```
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

附：

```
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


    @SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
    static int compareComparables(Class<?> kc, Object k, Object x) {
        return (x == null || x.getClass() != kc ? 0 :
                ((Comparable)k).compareTo(x));
    }

```

http://www.cnblogs.com/leesf456/p/5242233.html

http://blog.csdn.net/zerohuan/article/details/50351357

http://www.tuicool.com/articles/Yruqiye

http://www.importnew.com/7099.html