# 题目描述

运用你所掌握的数据结构，设计和实现一个 [LRU (最近最少使用) 缓存机制](https://baike.baidu.com/item/LRU)。它应该支持以下操作： 获取数据 `get` 和 写入数据 `put` 。

获取数据 `get(key)` - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 `put(key, value)` - 如果密钥已经存在，则变更其数据值；如果密钥不存在，则插入该组「密钥/数据值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

 

**进阶:**

你是否可以在 **O(1)** 时间复杂度内完成这两种操作？

 

**示例:**

```fortran
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得密钥 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得密钥 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```



# 解题思路

首先想到，使用哈希表来确定 key value 值，在该题中需要使用 $O(1)$ 的时间复杂度,所以想到可以首尾链表,将使用过的放在链表头部,未使用的逐渐向链表尾部移动. 如果到达缓存上限时直接删除尾部节点即可



代码

```go
// LRUCache 缓存机制

// LRUCache 定义缓存机制
type LRUCache struct {
    // 用于存储每个缓存的 key value 值
	cache   map[int]*DlinkNode
    // 表示该 LRU 的大小
	size 	int
    // 容量
	capacity int
    // 首尾双向链表,用于存储 (最近最少使用)
	head, tail *DlinkNode
}

type DlinkNode struct {
	key, value int
	prev, next *DlinkNode
}

func initDLinkNode(key, value int) *DlinkNode {
	return &DlinkNode{
		key: key,
		value: value,
	}
}


// Constructor is a constructor of LRUCache
func Constructor(capacity int) LRUCache {
    // 创建虚拟头部和虚拟尾部,标定界限，就不用判断周围是否存在节点
	l := LRUCache {
		cache: map[int]*DlinkNode{},
		head:  initDLinkNode(0, 0),
		tail:  initDLinkNode(0, 0),
		capacity: capacity,
	}
	l.head.next = l.tail
	l.tail.prev = l.head
	return l
}

// Get the value of key
func (this *LRUCache) Get(key int) int {
	if _, ok := this.cache[key]; !ok {
		return -1
	}
	node := this.cache[key]
    // 将其送入头部,表示最近使用过
	this.moveToHead(node)
	return node.value
}

// Put the key and value into LRUCache
func (this *LRUCache) Put(key int, value int) {
	if _, ok := this.cache[key]; !ok {
        // 如果不存在则插入并检查插入后 LRU 大小,并删除链表尾部节点(使用最少)
		node := initDLinkNode(key, value)
		this.cache[key] = node
		this.addToHead(node)
		this.size++
		if this.size > this.capacity {
			remove := this.removeTail()
			delete(this.cache, remove.key)
			this.size--
		}
	} else {
		node := this.cache[key]
		node.value = value
		this.moveToHead(node)
	}

}

// 将节点添加至头部
func (this *LRUCache) addToHead(node *DlinkNode) {
	node.prev = this.head
	node.next = this.head.next
	this.head.next.prev = node
	this.head.next = node
}

func (this *LRUCache) removeNode(node *DlinkNode) {
	node.prev.next = node.next
	node.next.prev = node.prev
}

func (this *LRUCache) moveToHead(node *DlinkNode) {
	this.removeNode(node)
	this.addToHead(node)
}

func (this *LRUCache) removeTail() *DlinkNode {
	node := this.tail.prev
	this.removeNode(node)
	return node
}
```

