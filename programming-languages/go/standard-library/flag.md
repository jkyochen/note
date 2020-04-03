# flag

## 在运行命令源码文件的时候传入参数，又怎样查看参数的使用说明

```golang
var name string

// 直接返回一个已经分配好的用于存储命令参数值的地址
// var name = flag.String("name", "everyone", "The greeting object.")

func init() {
    flag.StringVar(&name, "name", "everyone", "The greeting object.")
}

func main() {
    flag.Parse()
    fmt.Printf("Hello, %s!\n", name)
}
```

```sh
go run main.go -name="Robert"

go run main.go --help

Usage of /var/folders/xc/stqmg135375grtt1pdnlrz6c0000gn/T/go-build815513607/b001/exe/main:
  -name string
        The greeting object. (default "everyone")
exit status 2



go build main.go

./main --help

Usage of ./main:
  -name string
        The greeting object. (default "everyone")
```

## 自定义命令源码文件的参数使用说明

```golang
var name string

func init() {
    flag.StringVar(&name, "name", "everyone", "The greeting object.")
}

func main() {

    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage of %s:\n", "question")
        flag.PrintDefaults()
    }

    flag.Parse()
    fmt.Printf("Hello, %s!\n", name)
}
```

注意，对 flag.Usage 的赋值必须在调用 flag.Parse 函数之前。

## flag.ExitOnError / flag.PanicOnError

```golang
var name string

func init() {
    // flag.CommandLine = flag.NewFlagSet("", flag.ExitOnError)
    flag.CommandLine = flag.NewFlagSet("", flag.PanicOnError)
    flag.CommandLine.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage of %s:\n", "question")
        flag.PrintDefaults()
    }
    flag.StringVar(&name, "name", "everyone", "The greeting object.")
}

func main() {
    flag.Parse()
    fmt.Printf("Hello, %s!\n", name)
}
```

flag.ExitOnError 的含义是，告诉命令参数容器，当命令后跟 --help 或者参数设置的不正确的时候，在打印命令参数使用说明后以状态码 2 结束当前程序。状态码 2 代表用户错误地使用了命令。

flag.PanicOnError 与 flag.ExitOnError 的区别是在最后抛出“运行时恐慌（panic）”。上述两种情况都会在我们调用flag.Parse函数时被触发。

## 创建一个私有的命令参数容器。

```golang
var name string
var cmdLine = flag.NewFlagSet("question", flag.ExitOnError)

func init() {
    cmdLine.StringVar(&name, "name", "everyone", "The greeting object.")
}

func main() {
    cmdLine.Parse(os.Args[1:])
    fmt.Printf("Hello, %s!\n", name)
}
```

这样做的好处依然是更灵活地定制命令参数容器。但更重要的是，你的定制完全不会影响到那个全局变量 flag.CommandLine 。
