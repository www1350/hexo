---
title: 'HashMap源码(jdk1.8) '
abbrlink: afadb629
date: 2018-04-03 22:58:47
tags: HashMap
categories: 源码
---

- HashMap的底层主要是基于数组和链表来实现的，它之所以有相当快的查询速度主要是因为它是通过计算散列码来决定存储的位置。HashMap中主要是通过key的hashCode来计算hash值的，只要hashCode相同，计算出来的hash值就一样。如果存储的对象对多了，就有可能不同的对象所算出来的hash值是相同的，这就出现了所谓的hash冲突。学过数据结构的同学都知道，解决hash冲突的方法有很多，HashMap底层是通过链表来解决hash冲突的。
- JDK1.6中HashMap采用的是位桶+链表的方式，即我们常说的散列链表的方式，而JDK1.8中采用的是位桶+链表/红黑树的方式，也是非线程安全的。当某个位桶的链表的长度达到某个阀值的时候，这个链表就将转换成红黑树。也就是说原来如果hash不理想，所有都落入同一个桶就变成的单链表，现在只要大于8个在同一个桶里面，就会转化为红黑树，提升效率。
- 原来jdk1.7中resize是通过链表头插入的，jdk1.8是通过链表尾插入。可能有人会觉得1.7这样会更快，但是这样容易产生hash碰撞。如果它知道我们用的是哈希算法，它可能会发送大量的请求，导致产生严重的哈希碰撞。然后不停的访问这些 key就能显著的影响服务器的性能，这样就形成了一次拒绝服务攻击（DoS）

<!-- more -->

![image](https://cloud.githubusercontent.com/assets/7789698/18426598/702f1838-78f5-11e6-80ff-47cba88423ed.png)

解释下比较重要的几个变量：

loadFactor： 加载因子，默认是0.75

threshold：临界值，元素数量大于这个就resize。第一次resize的时候默认16*0.75截取int，其他时候每次扩容都扩大一倍，超过2^32取2^32。

table: 存储实体数组，类型是Node<K,V>

```java
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
//如16，则
// n|= 15 >>> 1   01111    --移位--> 00111      01111 | 00111   = 01111 (15)
// n|= 15 >>> 2   01111    --移位--> 00011      01111 | 00011   = 01111 (15)
// n|= 15 >>> 4   01111    --移位--> 00000      01111 | 00000   = 01111 (15)
// n|= 15 >>> 8
// n|= 15 >>> 16
//得 threshold 为16


//如15，则
// n|= 14 >>> 1   01110    --移位--> 00111      01110 | 00111   = 01111 (15)
// n|= 15 >>> 2   01111    --移位--> 00011      01111 | 00011   = 01111 (15)
// n|= 15 >>> 4   01111    --移位--> 00000      01111 | 00000   = 01111 (15)
// n|= 15 >>> 8
// n|= 15 >>> 16
//得 threshold 为16

//如19,则
// n|= 18 >>> 1   10010    --移位--> 01001      10010 | 01001   = 11011 (27)
// n|= 27 >>> 2   11011    --移位--> 00110      11011 | 00110   = 11111 (31)
// n|= 31 >>> 4   11111    --移位--> 00001      11111 | 00001   = 01111 (31)
// n|= 31 >>> 8
// n|= 31 >>> 16
//得 threshold 为32
//最后得能往2^n靠
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

```java
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
        
       static final int hash(Object key) {
        	int h;
        	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    	}
    }
```

(h = key.hashCode()) ^ (h >>> 16)

![image](https://user-images.githubusercontent.com/7789698/39800028-0b5f960a-5399-11e8-9923-aaf6cff8d72f.png)

当计算到hash值的时候可以轻松计算出位置


```java
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

```java
    final Node<K,V>[] resize() {
        //原数组
        Node<K,V>[] oldTab = table;
        //原数组大小
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //临界值，默认0
        int oldThr = threshold;
        //newCap新的数组大小，newThr新的临界值
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
 // 桶数组为空，首次分配，结合不同构造器的情况细节稍有不同：
        //构造器带初始大小的参数的，为比初始参数大的往2^n靠的数
        else if (oldThr > 0)
            newCap = oldThr;
        else {               //默认构造器构造的，使用默认值,newCap:16,newThr:12
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
		//无元素，带参数构造首次分配的时候，新的临界值设置成newCap * loadFactor取整
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //新的大小作为新的临界值
        threshold = newThr;
        //构造新数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
		//旧数组不为空
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
					//如果只有一个元素，直接rehash并放入桶中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //如果是红黑树进行拆分
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
                            //原索引
                            if ((e.hash & oldCap) == 0) {
                                //第一次, loTail一定为空，则loHead 和 loTail 都指向了e
                                if (loTail == null)
                                    loHead = e;
                                else
                                    //loTail不断指向新元素来达到添加的作用
                                    loTail.next = e;
                                loTail = e;
                            }
                            //原索引+oldCap
                            else {
                                //第一次, hiTail一定为空，hiTail 和 hiHead 都指向了e
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                     //hiTail不断指向新元素来达到添加的作用
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            //放入原来那个坑
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            //放到 原索引+oldCap的坑
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

其中：

1.每次扩容都是按2倍扩容

2.假如原来元素在5的位置，原来容量为16，临界值为12，扩容后容量为32。

​	（1）e.hash & (newCap - 1)   = e.hash % (newCap)

![image](https://user-images.githubusercontent.com/7789698/39800107-574e5cae-5399-11e8-9afb-f4d02e6dd506.png)

(a) 指原来的  (b)指扩容后的

可以明显看到可能出现的两种结果，一种和原来保持一致，一种就到新的位置上去了，因为扩容两倍，原来在0101现在就在10101了。

​	（2）e.hash & oldCap = e.hash % (oldCap -1)

![image](https://user-images.githubusercontent.com/7789698/39800402-2c02219c-539a-11e8-8f0b-e0550c5f9d98.png)

这样的原因是原来使用e.hash & (oldCap - 1)   已经把低位给hash过了，现在只要区分高位的数据是0还是1，放置的时候也只要 newTab[原索引 + oldCap] 

LinkedHashMap里面的Entry很神奇的（如何实现有序的hashmap，其实就是在hashmap的Entry加入前后指针）。

```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```


```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            //拿到桶中对应位置的首节点
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //不是首节点，往下取
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

```java
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