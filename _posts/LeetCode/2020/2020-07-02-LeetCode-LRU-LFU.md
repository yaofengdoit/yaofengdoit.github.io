---
layout: post
title: LRU、LFU 缓存设计
category: 算法
tags: [算法]
---

- LRU（Least Recently Used 最近最少使用）缓存，按照访问的时序来淘汰       
- LFU（最不经常使用）缓存，按照访问的频率来淘汰

这两个缓存设计问题对应leetcode 146 和 leetcode 460

大公司非常爱考察这两个问题，特别是亚马逊和头条。

## 1、LRU
两个动作：
- get(key)   如果key存在于缓存中，那么获取key的值，否则返回-1；
- put(key, value)   如果关键字已经存在，那么变更其值，如果不存在，那么存入该键-值对。当缓存容量达到上限时，在写入新数据之前删除最久未使用的数据值，为新的数据值留出空间。


思路：
- 容量满了删除最后一个数据，每次访问要把数据插入到队头；	
- cache结构应该满足：查找快、插入快、删除快、有顺序之分；
- 哈希表查找快，但是无序；链表有序，插入、删除快，但查找慢；两者结合一下->哈希链表

为什么必须要用双向链表，因为我们需要删除操作。删除一个节点不光要得到该节点本身的指针，也需要操作其前驱节点的指针，而双向链表才能支持直接查找前驱，保证操作的时间复杂度 O(1)。

在双向链表的实现中，使用一个伪头部（dummy head）和伪尾部（dummy tail）标记界限，这样在添加节点和删除节点的时候就不需要检查相邻的节点是否存在。

leetcode 146:
```java
class LRUCache {

    class DummyLinkedNode {
        private int key;
        private int value;
        DummyLinkedNode pre;
        DummyLinkedNode next;
        public DummyLinkedNode() {
        }
        public DummyLinkedNode(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private Map<Integer, DummyLinkedNode> cache = new HashMap<>();
    private int size;
    private int capacity;
    private DummyLinkedNode head, tail;

    public LRUCache(int capacity) {
        this.size = 0;
        this.capacity = capacity;
        // 虚拟头部和尾部
        head = new DummyLinkedNode();
        tail = new DummyLinkedNode();
        head.next = tail;
        tail.pre = head;
    }
    
    public int get(int key) {
        DummyLinkedNode node = cache.get(key);
        if (node == null) {
            return -1;
        }
        // key存在，先通过哈希表定位，再移到头部
        moveToHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        DummyLinkedNode node = cache.get(key);
        if (node == null) {
            // key不存在，那么创建一个新的节点
            DummyLinkedNode newNode = new DummyLinkedNode(key, value);
            cache.put(key, newNode);
            addToHead(newNode);
            size++;
            if (size > capacity) {
                // 超出容量，删除链表的尾部节点
                DummyLinkedNode temp = removeTail();
                // 删除哈希表中的对应项
                cache.remove(temp.key);
                size--;
            }
        } else {
            // key存在，先通过哈希表定位，再修改value，并移到头部
            node.value = value;
            moveToHead(node);
        }
    }

    private void removeNode(DummyLinkedNode node) {
        node.pre.next = node.next;
        node.next.pre = node.pre;
    }

    private void addToHead(DummyLinkedNode node) {
        node.pre = head;
        node.next = head.next;
        head.next.pre = node;
        head.next = node;
    }

    private void moveToHead(DummyLinkedNode node) {
        removeNode(node);
        addToHead(node);
    }

    private DummyLinkedNode removeTail() {
        // 注意tail是虚拟尾结点
        DummyLinkedNode res = tail.pre;
        removeNode(res);
        return res;
    }

}
```


## 2、LFU
