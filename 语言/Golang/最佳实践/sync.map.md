> [源码解读 Golang 的 sync.Map 实现原理](https://www.cnblogs.com/zkqiang/p/12551611.html)

## 简介

Go 的内建 map 是不支持并发写操作的，原因是 map 写操作不是并发安全的，当你尝试多个 Goroutine 操作同一个 map, 会产生报错：`fatal error: concurrent map writes`。

因此官方另外引入了 sync.map 来满足并发编程的引用。

`sync.map` 的实现原理可概括为：

- 通过 read 和 dirty 两个字段将读写分离，读的数据存在只读字段 read 中，将新写入的数据存在 dirty 字段上
- 读取时会先查询 read, 不存在再查询dirty，写入时则只写入 dirty
- 读取 read 并不需要加锁，而读或写 dirty 都需要加锁
- 另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据同步到 read 上
- 对于删除数据则直接通过标记来延迟删除。

## 数据结构

Map 的数据结构如下：

```go
type Map struct {
  // 加锁作用，保护 dirty 字段
	mu Mutex
  // 只读的数据，实际数据类型为 readonly 
  read atomic.Value
  // 最新写入的数据
  dirty map[interface{}]*entry
  // 计数器，每次需要读 dirty 则 +1 
  misses int
}
```

其中 readonly 数据结构为：

```go
type readonly struct {
  // 内建 map
  m map[interface{}]*entry
  // 表示 dirty 里存在 read 里没有的 key, 通过该字段决定是否加锁读 dirty
  amended bool
}
```

entry 数据结构则用于存储值的指针：

```go
type entry struct {
  p unsafe.Pointer  // 等同于 *interface{}
}
```

属性 p 有三种状态：

- `p == nil`: 键值已经被删除，且 `m.dirty ==nil`
- `p == expunged`: 键值已经被删除，但 `m.dirty != nil` 且 `m.dirty` 不存在键值（expunged 实际是空接口指针）
- 除以上情况，则键值对存在，存在于 `m.read.m` 中，如果 `m.dirty != nil` 则也存在于 `m.dirty`

Map 常用的有以下方法：

- `Load`: 读取指定 key 返回 value
- `Store`: 存储（增或改）key-value
- `Delete`: 删除指定 key 

## 源码解析



