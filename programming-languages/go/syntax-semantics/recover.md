# recover

恢复 panic，或者说平息运行时恐慌。

recover 函数无需任何参数，并且会返回一个空接口类型的值，如果在调用它时并没有 panic 发生，那么这个结果值就会是 nil。

如果被恢复的 panic 是我们通过调用 panic 函数引发的，那么它返回的结果值就会是我们传给 panic 函数参数值的副本。

recover 函数调用的返回值和 panic 函数的输入参数类型一致。

recover 函数捕获的是祖父一级调用函数栈帧的异常（刚好可以跨越一层 defer 函数）！

当希望将捕获到的异常转为错误时，如果希望忠实返回原始的信息，需要针对不同的类型分别处理：

```golang
func foo() (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch x := r.(type) {
            case string:
                err = errors.New(x)
            case error:
                err = x
            default:
                err = fmt.Errorf("Unknown panic: %v", r)
            }
        }
    }()

    panic("TODO")
}
```
