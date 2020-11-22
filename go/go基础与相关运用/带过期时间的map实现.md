# 过期时间 Map 

在 Redis 数据库中，key 的过期策略主要是：

1. 被动删除【惰性】：当 key 过期时，不进行删除，当实际访问的时候，判断是否过期，再采取措施
2. 主动删除：当 key 过期时，立即将 key 删除掉
3. 过量删除：当内存超过某个限制时，触发删除过期 key 的策略

被动删除，可以节省 CPU 性能，不用实时去判断删除 key，只需要在使用时访问判断即可，缺点是不能及时删除过期 key，内存会占用比较大。

主动删除，需要后台守护进程不断去判断并删除过期 key，比较耗费 CPU 性能，当数据量大时，可以有效节省内存。



实现时主要采用主动被动结合的形式，后台创建 goroutine 扫描检测是否有 key 过期，然后将其删除，目前设计删除精确时间为秒。



# 实现

## 结构体定义

```go
// val 定义 map 数据类型包含 data 和过期时间 expiredTime
type val struct {
	data        interface{}
	expiredTime int64
}

// ExpiredMap 定义过期时间 map 结构体
type ExpiredMap struct {
	// m 为存储数据 map
	m       map[interface{}]*val
	// timeMap 存储一个时间节点过期的 keys
	timeMap map[int64][]interface{}
	// 互斥锁
	lck     *sync.Mutex
	stop    chan struct{}
	// 置 1 时停止定义检查 goroutine
	needStop int32
}
```

结构体定义了带过期时间的值 `val` 和过期时间 map 的 `ExpiredMap` ，`ExpiredMap` 中主要两个值为 `m` 和 `timeMap` 两个值。

## 构造函数

```go
// NewExpiredMap 初始化构造函数
// return *ExpiredMap
func NewExpiredMap() *ExpiredMap {
	e := ExpiredMap{
		m:       make(map[interface{}]*val),
		lck:     new(sync.Mutex),
		timeMap: make(map[int64][]interface{}),
		stop:    make(chan struct{}),
	}
	atomic.StoreInt32(&e.needStop, 0)
	go e.run(time.Now().Unix())
	return &e
}
```

在构造函数中，储存了一个原子值，原子值表示是否需要停止 `run` 函数。同时启动了一个 `goroutine` 用于扫描 map 中过期的 key 并将其批量删除。

## Run 函数

```go
// run 后台守护进程，在初始化 Map 时启动一个 goroutine
// 循环检查是否有 key 存在过期，有则批量删除
func (e *ExpiredMap) run(now int64) {
	t := time.NewTicker(time.Second * 1)
	delCh := make(chan *delMsg, delChannelCap)
	go func() {
		for v := range delCh {
			if atomic.LoadInt32(&e.needStop) == 1 {
				fmt.Println("--- del stop ---")
				return
			}
			e.multiDelete(v.keys, v.t)
		}
	}()

	for {
		select {
		case <-t.C:
            // now++ 防止时间跳过 1s
			now++
			if keys, ok := e.timeMap[now]; ok {
				delCh <- &delMsg{keys: keys, t: now}
			}
		case <-e.stop:
			fmt.Println("==Stop==")
			atomic.StoreInt32(&e.needStop, 1)
			delCh <- &delMsg{keys: []interface{}{}, t: 0}
			return
		}
	}
}
```

在 `run` 中，主要为一个定期删除的 `gorotine` 和一个循环检测过期时间到  `channel` 中的主循环。

主循环中主要接受定时器 `ticker` 心跳包，每秒进行一次检测 map 中是否有过期的 key，若接受到 `stop` channel 信号则将原子值 `needStop` 置 1，将 `run` 协程关闭。



## Set、Get 方法

```go
func (e *ExpiredMap) Set(key, value interface{}, ttl int64) {
	if ttl <= 0 {
		return
	}
    // 保证多线程原子性
	e.lck.Lock()
	defer e.lck.Unlock()
    // 计算过期时间，将其存入 map 中
	expiredTime := time.Now().Unix() + ttl
	e.m[key] = &val{
		data:        value,
		expiredTime: expiredTime,
	}
    // 将该过期时间下的 key 存入 timeMap 中
	e.timeMap[expiredTime] = append(e.timeMap[expiredTime], key)
}

func (e *ExpiredMap) Get(key interface{}) (found bool, value interface{}) {
	e.lck.Lock()
	defer e.lck.Unlock()
    // 查询是否已被删除
	if found = e.checkDeleteKey(key); !found {
		return
	}
	value = e.m[key].data
	return
}
```

Set，Get 方法用于对 map 中进行插入值和查询值。

## 常规方法

```go
// 删除 map 中的某个 key
func (e *ExpiredMap) Delete(key interface{}) {
	e.lck.Lock()
	delete(e.m, key)
	e.lck.Unlock()
}

// 封装 Delete 方法
func (e *ExpiredMap) Remove(key interface{}) {
	e.Delete(key)
}

// 批量删除某个到期时间的 keys，以及将 timeMap 中该过期时间的所有 keys 删除
func (e *ExpiredMap) multiDelete(keys []interface{}, t int64) {
	e.lck.Lock()
	defer e.lck.Unlock()
	delete(e.timeMap, t)
	for _, key := range keys {
		delete(e.m, key)
	}
}
// 获取长度，不准确【可能会有过期的 key 在其中】
func (e *ExpiredMap) length() int {
	e.lck.Lock()
	defer e.lck.Unlock()
	return len(e.m)
}

func (e *ExpiredMap) Size() int {
	return e.length()
}
// TTL 获取 key 的剩余持续时间
func (e *ExpiredMap) TTL(key interface{}) int64 {
	e.lck.Lock()
	defer e.lck.Unlock()
	if !e.checkDeleteKey(key) {
		return -1
	}
	return e.m[key].expiredTime - time.Now().Unix()
}
// 清空 map 中所包含的 key
func (e *ExpiredMap) Clear() {
	e.lck.Lock()
	defer e.lck.Unlock()
	e.m = make(map[interface{}]*val)
	e.timeMap = make(map[int64][]interface{})
}
// 关闭 run 方法
func (e *ExpiredMap) Close() {
	e.lck.Lock()
	defer e.lck.Unlock()
	e.stop <- struct{}{}
}

func (e *ExpiredMap) Stop() {
	e.Close()
    e.Clear()
}
// 检测待删除 key
func (e *ExpiredMap) checkDeleteKey(key interface{}) bool {
	if val, ok := e.m[key]; ok {
		if val.expiredTime <= time.Now().Unix() {
			delete(e.m, key)
			return false
		}
		return true
	}
	return false
}
```

在其中定义了清除函数以及查询函数。

## Keys 操作方法

```go
func (e *ExpiredMap) DoForEach(handler func(interface{}, interface{})) {
	e.lck.Lock()
	defer e.lck.Unlock()
	for k, v := range e.m {
		if !e.checkDeleteKey(k) {
			continue
		}
		handler(k, v)
	}
}
// foreach 方法执行一次后退出
func (e *ExpiredMap) DoForEachWithBreak(handler func(interface{}, interface{}) bool) {
	e.lck.Lock()
	defer e.lck.Unlock()
	for k, v := range e.m {
		if !e.checkDeleteKey(k) {
			continue
		}
		if handler(k, v) {
			break
		}
	}
}
```



