# context

context.Context 类型（以下简称 Context 类型）是在 Go 1.7 发布时才被加入到标准库的。而后，标准库中的很多其他代码包都为了支持它而进行了扩展，包括：os/exec 包、net 包、database/sql 包，以及 runtime/pprof 包和 runtime/trace 包，等等。

它是一种非常通用的同步工具。它的值不但可以被任意地扩散，而且还可以被用来传递额外的信息和信号。

Context 类型可以提供一类代表上下文的值。此类值是并发安全的，也就是说它可以被传播给多个 goroutine。

## 怎样使用 context 包中的程序实体，实现一对多的 goroutine 协作流程？

```golang
package main

import (
    "context"
    "fmt"
    "sync/atomic"
    "time"
)

func main() {
    coordinateWithContext()
}

func coordinateWithContext() {
    total := 12
    var num int32
    fmt.Printf("The number: %d [with context.Context]\n", num)
    cxt, cancelFunc := context.WithCancel(context.Background())
    for i := 1; i <= total; i++ {
        go addNum(&num, i, func() {
            if atomic.LoadInt32(&num) == int32(total) {
                cancelFunc()
            }
        })
    }
    <-cxt.Done()
    fmt.Println("End.")
}

// addNum 用于原子地增加一次 numP 所指的变量的值。
func addNum(numP *int32, id int, deferFunc func()) {
    defer func() {
        deferFunc()
    }()
    for i := 0; ; i++ {
        currNum := atomic.LoadInt32(numP)
        newNum := currNum + 1
        time.Sleep(time.Millisecond * 200)
        if atomic.CompareAndSwapInt32(numP, currNum, newNum) {
            fmt.Printf("The number: %d [%d-%d]\n", newNum, id, i)
            break
        } else {
            fmt.Printf("The CAS operation failed. [%d-%d]\n", id, i)
        }
    }
}
```

由于一旦 cancelFunc 函数被调用，针对该通道的接收操作就会马上结束，所以，这样做就可以实现“等待所有的 addNum 函数都执行完毕”的功能。

正因为如此，所有的 Context 值共同构成了一颗代表了上下文全貌的树形结构。这棵树的树根（或者称上下文根节点）是一个已经在 context 包中预定义好的 Context 值，它是全局唯一的。通过调用 context.Background 函数，我们就可以获取到它（我在 coordinateWithContext 函数中就是这么做的）。

除此之外，context 包中还包含了四个用于繁衍 Context 值的函数，即：WithCancel、WithDeadline、WithTimeout 和 WithValue。

WithCancel 函数用于产生一个可撤销的 parent 的子值。在 coordinateWithContext 函数中，我通过调用该函数，获得了一个衍生自上下文根节点的 Context 值，和一个用于触发撤销信号的函数。

而 WithDeadline 函数和 WithTimeout 函数则都可以被用来产生一个会定时撤销的 parent 的子值。至于 WithValue 函数，我们可以通过调用它，产生一个会携带额外数据的 parent 的子值。

### 问题 1：“可撤销的”在 context 包中代表着什么？“撤销”一个 Context 值又意味着什么？

Done 方法会返回一个元素类型为 struct{} 的接收通道。不过，这个接收通道的用途并不是传递元素值，而是让调用方去感知“撤销”当前 Context 值的那个信号。

一旦当前的 Context 值被撤销，这里的接收通道就会被立即关闭。我们都知道，对于一个未包含任何元素值的通道来说，它的关闭会使任何针对它的接收操作立即结束。

除了让 Context 值的使用方感知到撤销信号，让它们得到“撤销”的具体原因，有时也是很有必要的。后者即是 Context 类型的 Err 方法的作用。该方法的结果是 error 类型的，并且其值只可能等于 context.Canceled 变量的值，或者 context.DeadlineExceeded 变量的值。前者用于表示手动撤销，而后者则代表：由于我们给定的过期时间已到，而导致的撤销。

当我们通过调用 context.WithCancel 函数产生一个可撤销的 Context 值时，还会获得一个用于触发撤销信号的函数。通过调用这个函数，我们就可以触发针对这个 Context 值的撤销信号。一旦触发，撤销信号就会立即被传达给这个 Context 值，并由它的 Done 方法的结果值（一个接收通道）表达出来。

撤销函数只负责触发信号，而对应的可撤销的 Context 值也只负责传达信号，它们都不会去管后边具体的“撤销”操作。实际上，我们的代码可以在感知到撤销信号之后，进行任意的操作，Context 值对此并没有任何的约束。

最后，若再深究的话，这里的“撤销”最原始的含义其实就是，终止程序针对某种请求（比如 HTTP 请求）的响应，或者取消对某种指令（比如 SQL 指令）的处理。这也是 Go 语言团队在创建 context 代码包，和 Context 类型时的初衷。

### 问题 2：撤销信号是如何在上下文树中传播的？

在撤销函数被调用之后，对应的 Context 值会先关闭它内部的接收通道，也就是它的 Done 方法会返回的那个通道。然后，它会向它的所有子值（或者说子节点）传达撤销信号。这些子值会如法炮制，把撤销信号继续传播下去。最后，这个 Context 值会断开它与其父值之间的关联。

通过调用 context 包的 WithDeadline 函数或者 WithTimeout 函数生成的 Context 值也是可撤销的。它们不但可以被手动撤销，还会依据在生成时被给定的过期时间，自动地进行定时撤销。这里定时撤销的功能是借助它们内部的计时器来实现的。当过期时间到达时，这两种 Context 值的行为与 Context 值被手动撤销时的行为是几乎一致的，只不过前者会在最后停止并释放掉其内部的计时器。

最后要注意，通过调用 context.WithValue 函数得到的 Context 值是不可撤销的。撤销信号在被传播时，若遇到它们则会直接跨过，并试图将信号直接传给它们的子值。

### 问题 3：怎样通过 Context 值携带数据？怎样从中获取数据？

WithValue 函数在产生新的 Context 值（以下简称含数据的 Context 值）的时候需要三个参数，即：父值、键和值。与“字典对于键的约束”类似，这里键的类型必须是可判等的。原因很简单，当我们从中获取数据的时候，它需要根据给定的键来查找对应的值。

Context 类型的 Value 方法就是被用来获取数据的。在我们调用含数据的 Context 值的 Value 方法时，它会先判断给定的键，是否与当前值中存储的键相等，如果相等就把该值中存储的值直接返回，否则就到其父值中继续查找。

注意，除了含数据的 Context 值以外，其他几种 Context 值都是无法携带数据的。因此，Context 值的 Value 方法在沿路查找的时候，会直接跨过那几种值。

如果我们调用的 Value 方法的所属值本身就是不含数据的，那么实际调用的就将会是其父辈或祖辈的 Value 方法。这是由于这几种 Context 值的实际类型，都属于结构体类型，并且它们都是通过“将其父值嵌入到自身”，来表达父子关系的。

最后，提醒一下，Context 接口并没有提供改变数据的方法。因此，在通常情况下，我们只能通过在上下文树中添加含数据的 Context 值来存储新的数据，或者通过撤销此种值的父值丢弃掉相应的数据。如果你存储在这里的数据可以从外部改变，那么必须自行保证安全。

## 总结

Context 类型是一个可以帮助我们实现多 goroutine 协作流程的同步工具。不但如此，我们还可以通过此类型的值传达撤销信号或传递数据。

Context 类型的实际值大体上分为三种，即：根 Context 值、可撤销的 Context 值和含数据的 Context 值。所有的 Context 值共同构成了一颗上下文树。这棵树的作用域是全局的，而根 Context 值就是这棵树的根。它是全局唯一的，并且不提供任何额外的功能。

可撤销的 Context 值又分为：只可手动撤销的 Context 值，和可以定时撤销的 Context 值。

一旦撤销函数被调用，撤销信号就会立即被传达给对应的 Context 值，并由该值的 Done 方法返回的接收通道表达出来。

“撤销”这个操作是 Context 值能够协调多个 goroutine 的关键所在。撤销信号总是会沿着上下文树叶子节点的方向传播开来。

含数据的 Context 值可以携带数据。每个值都可以存储一对键和值。在我们调用它的 Value 方法的时候，它会沿着上下文树的根节点的方向逐个值的进行查找。如果发现相等的键，它就会立即返回对应的值，否则将在最后返回 nil。

含数据的 Context 值不能被撤销，而可撤销的 Context 值又无法携带数据。但是，由于它们共同组成了一个有机的整体（即上下文树），所以在功能上要比 sync.WaitGroup 强大得多。
