# Error

Go 语言中的错误是一种接口类型。接口信息中包含了原始类型和原始的值。只有当接口的类型和原始的值都为空的时候，接口的值才对应 nil。其实当接口中类型为空的时候，原始值必然也是空的；反之，当接口对应的原始值为空的时候，接口对应的原始类型并不一定为空的。

```golang
func echo(request string) (response string, err error) {
    if request == "" {
        err = errors.New("empty request")
        return
    }
    response = fmt.Sprintf("echo: %s", request)
    return
}

func main() {
    for _, req := range []string{"", "hello!"} {
        fmt.Printf("request: %s\n", req)
        resp, err := echo(req)
        if err != nil {
            fmt.Printf("error: %s\n", err)
            continue
        }
        fmt.Printf("response: %s\n", resp)
    }
}
```

使用 error 类型的方式通常是，在函数声明的结果列表的最后，声明一个该类型的结果，同时在调用这个函数之后，先判断它返回的最后一个结果值是否 “不为 nil”。

生成 error 类型值的时候用到了 `errors.New` 函数。这是一种最基本的生成错误值的方式。我们调用它的时候传入一个由字符串代表的错误信息，它会给返回给我们一个包含了这个错误信息的 error 类型值。

该值的静态类型当然是 error，而动态类型则是一个在 errors 包中的，包级私有的类型 `*errorString`。errorString 类型拥有的一个指针方法实现了 error 接口中的 Error 方法。这个方法在被调用后，会原封不动地返回我们之前传入的错误信息。实际上，error 类型值的 Error 方法就相当于其他类型值的 String 方法。

对于其他类型的值来说，只要我们能为这个类型编写一个 String 方法，就可以自定义它的字符串表示形式。而对于 error 类型值，它的字符串表示形式则取决于它的 Error 方法。

fmt 包中的打印函数如果发现被打印的值是一个 error 类型的值，那么就会去调用它的 Error 方法。

当我们想通过模板化的方式生成错误信息，并得到错误值时，可以使用 fmt.Errorf 函数。

## 对于具体错误的判断，Go 语言中都有哪些惯用法？ / 怎样判断一个错误值具体代表的是哪一类错误？

1. 对于类型在已知范围内的一系列错误值，一般使用类型断言表达式或类型 switch 语句来判断；
2. 对于已有相应变量且类型相同的一系列错误值，一般直接使用判等操作来判断；
3. 对于没有相应变量且类型未知的一系列错误值，只能使用其错误信息的字符串表示形式来做判断。

### 已知范围内的一系列错误值

```golang
func underlyingError(err error) error {
    switch err := err.(type) {
        case *os.PathError:
            return err.Err
        case *os.LinkError:
            return err.Err
        case *os.SyscallError:
            return err.Err
        case *exec.Error:
            return err.Err
    }
    return err
}

func main() {

    printError := func(i int, err error) {
        if err == nil {
            fmt.Println("nil error")
            return
        }
        err = underlyingError(err)
        switch err {
            case os.ErrClosed:
                fmt.Println("error(closed)[%d]: %s\n", i, err)
            case os.ErrInvalid:
                fmt.Println("error(invalid)[%d]: %s\n", i, err)
            case os.ErrPermission:
                fmt.Println("error(permission)[%d]: %s\n", i, err)
        }
    }
}
```

类型在已知范围内的错误值其实是最容易分辨的。就拿 os 包中的几个代表错误的类型 os.PathError、os.LinkError、os.SyscallError 和 os/exec.Error 来说，它们的指针类型都是 error 接口的实现类型，同时它们也都包含了一个名叫 Err，类型为 error 接口类型的代表潜在错误的字段。

如果我们得到一个 error 类型值，并且知道该值的实际类型肯定是它们中的某一个，那么就可以用类型 switch 语句去做判断。

### 类型相同的一系列错误值

不少的错误值都是通过调用 errors.New 函数来初始化的，比如：os.ErrClosed、os.ErrInvalid 以及 os.ErrPermission，等等。

如果我们在操作文件系统的时候得到了一个错误值，并且知道该值的潜在错误值肯定是上述值中的某一个，那么就可以用普通的 switch / if 语句去做判断。

### 类型未知的一系列错误值

如果我们对一个错误值可能代表的含义知之甚少，那么就只能通过它拥有的错误信息去做判断了。

总是能通过错误值的 Error 方法，拿到它的错误信息。其实 os 包中就有做这种判断的函数，比如：os.IsExist、os.IsNotExist 和 os.IsPermission。

## 问题：怎样根据实际情况给予恰当的错误值？

1. 创建立体的错误类型体系
2. 创建扁平的错误值列表

### 多层分类

树结构

### 链式关系

当我们细看 net 包中的这些具体错误类型的实现时，还会发现，与 os 包中的一些错误类型类似，它们也都有一个名为 Err、类型为 error 接口类型的字段，代表的也是当前错误的潜在错误。

使用者调用 net.DialTCP 之类的函数时，net 包中的代码可能会返回给他一个 `*net.OpError`类型的错误值，以表示由于他的操作不当造成了一个错误。同时，这些代码还可能会把一个 `*net.AddrError` 或 `net.UnknownNetworkError` 类型的值赋给该错误值的 Err 字段，以表明导致这个错误的潜在原因。如果，此处的潜在错误值的 Err 字段也有非 nil 的值，那么将会指明更深层次的错误原因。如此一级又一级就像链条一样最终会指向问题的根源。

用类型建立起树形结构的错误体系，用统一字段建立起可追根溯源的链式错误关联。

不过要注意，如果你不想让包外代码改动你返回的错误值的话，一定要小写其中字段的名称首字母。你可以通过暴露某些方法让包外代码有进一步获取错误信息的权限，比如编写一个可以返回包级私有的 err 字段值的公开方法 Err。

由于 error 是接口类型，所以通过 `errors.New` 函数生成的错误值只能被赋给变量，而不能赋给常量，又由于这些代表错误的变量需要给包外代码使用，所以其访问权限只能是公开的。

先私有化此类变量，然后编写公开的用于获取错误值以及用于判等错误值的函数。

比如，对于错误值 os.ErrClosed，先改写它的名称，让其变成 os.errClosed，然后再编写 ErrClosed 函数和 IsErrClosed 函数。

## syscall 包

在 syscall 包中。该包中有一个类型叫做 Errno ，该类型代表了系统调用时可能发生的底层错误。这个错误类型是 error 接口的实现类型，同时也是对内建类型 uintptr 的再定义类型。

由于 uintptr 可以作为常量的类型，所以 `syscall.Errno` 自然也可以。syscall 包中声明有大量的 Errno 类型的常量，每个常量都对应一种系统调用错误。syscall 包外的代码可以拿到这些代表错误的常量，但却无法改变它们。
