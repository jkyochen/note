# func

函数类型属于**引用类型**，它的值可以为 nil，而这种类型的零值恰恰就是 nil。

```golang
type operate func(x, y int) int

func calculate(x int, y int, op operate) (int, error) {
    if op == nil {
        return 0, errors.New("invalid operation")
    }
    return op(x, y), nil
}

func main() {

    op := func(x, y int) int {
        return x + y
    }

    var rs, err = calculate(12, 12, op);
    fmt.Println(rs, err);
}
```

只要两个函数的参数列表和结果列表中的元素顺序及其类型是一致的，我们就可以说它们是一样的函数，或者说是实现了同一个函数类型的函数。

严格来说，函数的名称也不能算作函数签名的一部分，它只是我们在调用函数时，需要给定的标识符而已。
