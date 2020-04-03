# select

1. 有默认分支，那么无论涉及通道操作的表达式是否有阻塞，select 语句都不会被阻塞。如果那几个表达式都阻塞了，或者说都没有满足求值的条件，那么默认分支就会被选中并执行。

2. 没有加入默认分支，那么一旦所有的 case 表达式都没有满足求值条件，那么 select 语句就会被阻塞。直到至少有一个 case 表达式满足条件为止。

3. 由于通道关闭了，而从通道接收到一个其元素类型的零值。所以需要通过接收表达式的第二个结果值来判断通道是否已经关闭。一旦发现某个通道关闭了，我们就应该及时地屏蔽掉对应的分支或者采取其他措施。这对于程序逻辑和程序性能都是有好处的。

4. select 语句只能对其中的每一个 case 表达式各求值一次。所以，如果我们想连续或定时地操作其中的通道的话，就往往需要通过在 for 语句中嵌入 select 语句的方式实现。但这时要注意，简单地在 select 语句的分支中使用 break 语句，只能结束当前的 select 语句的执行，而并不会对外层的 for 语句产生作用。这种错误的用法可能会让这个 for 语句无休止地运行下去。

```golang
intChannels := [3]chan int{
    make(chan int, 1),
    make(chan int, 1),
    make(chan int, 1),
}

index := rand.Intn(3)
fmt.Printf("The index %d\n", index)

intChannels[index] <- index
select {
    case <- intChannels[0]:
        fmt.Println("The first candidate case is selected.")
    case <- intChannels[1]:
        fmt.Println("The second candidate case is selected.")
    case elem := <- intChannels[2]:
        fmt.Printf("The third candidate case is selected, the element is %d.\n", elem)
    default:
        fmt.Println("No candidate case is selected!")
}

intChan := make(chan int, 1)
time.AfterFunc(time.Second, func() {
    close(intChan)
})

select {
case _, ok := <-intChan:
    if !ok {
        fmt.Println("The candidate case is closed.")
        break
    }
    fmt.Println("The candidate case is selected.")
}
```
