本节主要讨论`Go`的并发编程

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

# lock锁

## 🚩零值的锁结构都是开箱即用的
**zero-valus of lock is valid**
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```

</td><td>

```go
var mu sync.Mutex
mu.Lock()
```
</td></tr>
</tbody></table>

虽然`Lock`、`Unlock`是接受者为`pointer`的方法，但是仍然可以直接用`var mu = sync.Mutex`
```go
func (m *Mutex) Lock()
func (m *Mutex) Unlock() 
```
理由见`method`章节的`value receiver & pointer receiver`

## 🚩当心 Mutex 的错误复制
todo
[Beware of copying mutexes in Go](https://eli.thegreenplace.net/2018/beware-of-copying-mutexes-in-go/)

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Container struct {
  sync.Mutex                       // <-- Added a mutex
  counters map[string]int
}

func (c Container) inc(name string) {
  c.Lock()                         // <-- Added locking of the mutex
  defer c.Unlock()
  c.counters[name]++
}

func main() {
  c := Container{counters: map[string]int{"a": 0, "b": 0}}

  doIncrement := func(name string, n int) {
    for i := 0; i < n; i++ {
      c.inc(name)
    }
  }

  go doIncrement("a", 100000)
  go doIncrement("a", 100000)

  // Wait a bit for the goroutines to finish
  time.Sleep(300 * time.Millisecond)
  fmt.Println(c.counters)
}
```

直接调用上述程序是会报错的，但是只需要修改一个字符，就能完美运行。

你知道如何做吗？

# goroutine

## 关于 goroutine 应该知道的知识
* 只要`main goroutine`退出，不管其余`children goroutine`是否运行完毕，直接退出
* 如果某个`goroutine`在函数/方法的调用时出现`panic`，一个被称之为`panicking`的过程被激活，一直向上，运行上层每一层的`defer`（如果有的话），如果没有使用`recover`捕获该`panic`，那么整个程序会异常退出。


## 🚩Before you launch a goroutine, know when it will stop


# channel
## 🌵To pass a signal prefer chan struct{} instead of chan bool.
```go
type Service struct {
	deleteCh chan bool // what does this bool mean? 
}

type Service struct {
	deleteCh chan struct{} // ok, if event than delete something.
}
```
## 关于特殊状态的 channel ，需要注意以下事实


|       | closed channel | nil channel |
|-------|----------------|-------------|
| ch <- | panic          | 阻塞          |
| <- ch | channel 中元素的零值 | 阻塞          |


## 🚩channel 的大小要么是 1 ，要么是 0
TODO


# 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Beware of copying mutexes in Go](https://eli.thegreenplace.net/2018/beware-of-copying-mutexes-in-go/)