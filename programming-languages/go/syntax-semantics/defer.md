# defer

```golang
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

defer 语句就是被用来延迟执行代码的。延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。这里存在一些限制，有一些调用表达式是不能出现在这里的，包括：针对 Go 语言内建函数的调用表达式，以及针对 unsafe 包中的函数的调用表达式。对于 go 语句中的调用表达式，限制也是一样的。

在这里被调用的函数可以是有名称的，也可以是匿名的。我们可以把这里的函数叫做 defer 函数或者延迟函数。被延迟执行的是 defer 函数，而不是 defer 语句。

我们要尽量把 defer 语句写在函数体的开始处，因为在引发 panic 的语句之后的所有语句，都不会有任何执行机会。

## 多条 defer 语句函数调用的执行顺序

在同一个函数中，defer 函数调用的执行顺序与它们分别所属的 defer 语句的出现顺序（执行顺序）完全相反。

在 defer 语句每次执行的时候，Go 语言会把它携带的 defer 函数及其参数值另行存储到一个队列中。这个队列与该 defer 语句所属的函数是对应的，并且，它是先进后出（FILO）的，相当于一个栈。在需要执行某个函数中的 defer 函数调用的时候，Go 语言会先拿到对应的队列，然后从该队列中一个一个地取出 defer 函数及其参数值，并逐个执行调用。
