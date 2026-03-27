## Goroutine
常用：  `go func() {} ()` 来开启一个协程
例如：
```go func main() {
    sum := 0
    for range 10 {
        go func() {
            for range 100000 {
                sum += 1
            }
        }()
    }
    time.Sleep(1 * time.Second)
    fmt.Println(sum)
    }
```
## 互斥锁

### "sync/atomic"

直接上例子
```go
package main

import (
    "fmt"
    "sync/atomic"
    "time"
)

func main() {
    sum := atomic.Int64{}
    sum.Store(0)
    for range 10 {
        go func() {
            for range 100000 {
                sum.Add(1)
            }
        }()
    }
    time.Sleep(1 * time.Second)
    fmt.Println(sum.Load())
}
```
直接进行原子操作

### “sync”

#### Mutex

上例子来
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    sum := 0
    // 互斥锁
    lock := sync.Mutex{}
    for range 10 {
        go func() {
            for range 100000 {
                lock.Lock()
                sum += 1
                lock.Unlock()
            }
        }()
    }
    time.Sleep(1 * time.Second)
    fmt.Println(sum)
}
```
通过加锁来保护变量

#### WaitGroup

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    wg := sync.WaitGroup{}
    for range 10 {
        // wg.Add 代表添加一个执行任务
        wg.Add(1)
        go func() {
            fmt.Println(1)
            // wg.Done() 代表执行完成
            wg.Done()
        }()
    }
    // 等待 Add 的任务全部 Done
    wg.Wait()
}
```

## Channel

```go
package main

import (
    "fmt"
    "time"
)

type Task struct {
    // 函数也可以是结构体的成员
    Runnable func(workerId int)
}

func main() {
    // 一个负责任务分发的管道
    ch := make(chan Task, 10)

    // 启动几个 worker 负责处理任务
    for id := range 10 {
        go func(workerId int) {
            for t := range ch {
                t.Runnable(workerId)
            }
        }(id)
    }

    // 任务分发
    for i := range 20 {
	    j := i
        t1 := Task{
            Runnable: func(workerId int) {
                fmt.Printf("workerId%v：task%v做一件事情\n", workerId, j)
            },
        }
        ch <- t1
    }
    time.Sleep(1 * time.Second)
    close(ch)
}
```

