# pointer

## Go 语言中的哪些值是不可寻址的吗？

1. 不可变的值不可寻址。常量、基本类型的值字面量、字符串变量的值、函数以及方法的字面量都是如此。其实这样规定也有安全性方面的考虑。

2. 绝大多数被视为临时结果的值都是不可寻址的。算术操作的结果值属于临时结果，针对值字面量的表达式结果值也属于临时结果。但有一个例外，对切片字面量的索引结果值虽然也属于临时结果，但却是可寻址的。

3. 若拿到某值的指针可能会破坏程序的一致性，那么就是不安全的，该值就不可寻址。由于字典的内部机制，对字典的索引结果值的取址操作都是不安全的。另外，获取由字面量或标识符代表的函数或方法的地址显然也是不安全的。

## 问题 1：不可寻址的值在使用上有哪些限制？

无法使用取址操作符 & 获取它们的指针。

```golang
type Dog struct {
    name string
}

func (dog *Dog) setName(name string) {
    dog.name = name
}

func New(name string) Dog {
    return Dog{name}
}
```

`New("little pig").SetName("monster")`，调用New函数所得到的结果值属于临时结果，是不可寻址的，所以无法对它进行取址操作。

### ++ / --

Go 语言中的 ++ 和 -- 并不属于操作符，而分别是自增语句和自减语句的重要组成部分。

虽然 Go 语言规范中的语法定义是，只要在 ++ 或 -- 的左边添加一个表达式，就可以组成一个自增语句或自减语句，但是这个表达式的结果值必须是可寻址的。

### 例外

1. 字典字面量和字典变量索引表达式的结果值都是不可寻址的，但是这样的表达式却可以被用在自增语句和自减语句中。

2. 在赋值语句中，赋值操作符左边的表达式的结果值必须可寻址的，但是对字典的索引结果值也是可以的。

3. 在 range 关键字左边的表达式的结果值也都必须是可寻址的，不过对字典的索引结果值同样可以被用在这里。

## 问题 2：怎样通过 unsafe.Pointer 操纵可寻址的值？

```golang
type Dog struct {
    name string
}

func (dog *Dog) setName(name string) {
    dog.name = name
}

func New(name string) Dog {
    return Dog{name}
}

func main() {
    dog := Dog{"little pig"}
    dogP := &dog
    dogPtr := uintptr(unsafe.Pointer(dogP))

    namePtr := dogPtr + unsafe.Offsetof(dogP.name)
    nameP := (*string)(unsafe.Pointer(namePtr))

    fmt.Println(dogPtr, namePtr, *nameP)
}
```

### 转换规则

1. 一个指针值（比如 `*Dog` 类型的值）可以被转换为一个 unsafe.Pointer 类型的值，反之亦然。
2. 一个 uintptr 类型的值也可以被转换为一个 unsafe.Pointer 类型的值，反之亦然。
3. 一个指针值无法被直接转换成一个 uintptr 类型的值，反过来也是如此。


只要有 namePtr 就可以。它是一个无符号整数，但同时也是一个指向了程序内部数据的内存地址。它可能会给我们带来一些好处，比如可以直接修改埋藏得很深的内部数据。
