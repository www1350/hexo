---
title: 线程安全队列Queue
tags: Queue
categories: Queue
abbrlink: a5db83c3
date: 2016-07-05 22:08:54
---

线程安全的Queue可以分为阻塞队列和非阻塞队列，其中阻塞队列的典型例子是BlockingQueue，非阻塞队列的典型例子是ConcurrentLinkedQueue

- 单生产者，单消费者  用 LinkedBlockingqueue
- 多生产者，单消费者   用 LinkedBlockingqueue
- 单生产者 ，多消费者   用 ConcurrentLinkedQueue
- 多生产者 ，多消费者   用 ConcurrentLinkedQueue

可能报异常 返回布尔值 可能阻塞    设定等待时间
- 入队    add(e)  offer(e)    put(e)  offer(e, timeout, unit)
- 出队    remove()    poll()  take()  poll(timeout, unit)
- 查看    element()   peek()  无 无
- add(e) remove() element() 方法不会阻塞线程。当不满足约束条件时，会抛出IllegalStateException 异常。例如：当队列被元素填满后，再调用add(e)，则会抛出异常。
- offer(e) poll() peek() 方法即不会阻塞线程，也不会抛出异常。例如：当队列被元素填满后，再调用offer(e)，则不会插入元素，函数返回false。
- 要想要实现阻塞功能，需要调用put(e) take() 方法。当不满足约束条件时，会阻塞线程。

ConcurrentLinkedQueue内部有个匿名内部类Node
源码如下

```
private static class Node<E> {
        volatile E item;
        volatile Node<E> next;

        /**
         * Constructs a new node.  Uses relaxed write because item can
         * only be seen after publication via casNext.
         */
        Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        void lazySetNext(Node<E> val) {
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        // Unsafe mechanics

        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```