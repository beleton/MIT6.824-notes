---
dg-publish: true
---
主要包括以下几种方法：
1. **互斥锁（Mutex）**
2. **读写锁（RWMutex）**
3. **等待组（WaitGroup）**
4. **条件变量（Cond）**
5. **通道（Channel）**

如果线程间不需要共享内存（变量等），通常使用channel完成线程间的通信。如果线程间需要共享内存，则采用锁+条件变量的方案。
### 1. 互斥锁（Mutex）

互斥锁用于确保在同一时间只有一个goroutine能够访问临界区代码，从而防止竞态条件。
```go
var mu sync.Mutex

mu.Lock()
// 临界区代码
mu.Unlock()
```

### 2. 读写锁（RWMutex）

读写锁允许多个读取操作同时进行，但写操作会独占锁。适用于读多写少的场景。
```go
var rwMu sync.RWMutex

rwMu.RLock()
// 读取操作
rwMu.RUnlock()

rwMu.Lock()
// 写入操作
rwMu.Unlock()
```

### 3. 等待组（WaitGroup）

等待组用于等待一组goroutine完成。在主goroutine中调用`WaitGroup`的`Wait`方法，阻塞直到所有goroutine都完成。
```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    // 执行任务
}()

wg.Wait()
```

### 4. 条件变量（Cond）

条件变量允许goroutine在特定条件满足时等待或被唤醒。通常与互斥锁结合使用。
```go
var mu sync.Mutex
cond := sync.NewCond(&mu)

mu.Lock()
for !condition {
    cond.Wait() //释放与条件变量相关的互斥锁，并等待唤醒
}
// 执行条件满足时的操作
cond.Signal() // 唤醒一个等待中的goroutine，被唤醒的goroutine等待当前线程释放锁，并重新获得互斥锁
mu.Unlock()
```
示例
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var (
    mu    sync.Mutex
    cond  = sync.NewCond(&mu)
    ready = false
)

func waitCondition() {
    mu.Lock()
    for !ready {
        cond.Wait()
    }
    fmt.Println("Condition met")
    mu.Unlock()
}

func setCondition() {
    mu.Lock()
    ready = true
    cond.Signal()
    mu.Unlock()
}

func main() {
    go waitCondition()
    time.Sleep(1 * time.Second) // 模拟一些工作
    setCondition()
    time.Sleep(1 * time.Second) // 确保输出显示
}
```
需要再持有锁的情况下调用Singal：
	
### 5. 通道（Channel）

通道用于在goroutine之间传递数据和同步操作，是Go语言的核心并发原语。
- 无缓冲通道：`make(chan Type)`，在发送和接收双方都准备好之前，会阻塞发送和接收操作。每次发送操作都必须有一个对应的接收操作，确保数据不会在通道中滞留。
- 有缓冲通道：`make(chan Type, capacity)`，当通道缓冲区满时，发送操作会阻塞；当通道缓冲区为空时，接收操作会阻塞。

通道的发送和接受可能导致阻塞，但条件变量的Singal()或Broadcast()方法并不是阻塞操作。


一些错误：
在`goroutine`中使用可能会被并发修改的外部变量时，应通过参数传递的方式将变量的副本传入`goroutine`
```go
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i) // 可能会打印出意外的值
    }()
}
```
对于以上代码，`goroutine`可能会在`for`循环结束后才开始执行，而此时变量`i`的值可能已经是循环结束时的最终值。应该将`i`作为参数传递给`goroutine`
```go
for i := 0; i < 10; i++ {
    go func(i int) {
        fmt.Println(i) // 将打印出预期的值
    }(i)
}
```
