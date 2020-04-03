# switch

普通 case 子句的编写顺序很重要，最上边的 case 子句中的子表达式总是会被最先求值，在判等的时候顺序也是这样。

## 问题 1：switch 语句中的 switch 表达式和 case 表达式之间有着怎样的联系？

```golang
value1 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch int8(1 + 3) {
    case value1[0], value1[1]:
        fmt.Println("0 or 1")
    case value1[2], value1[3]:
        fmt.Println("2 or 3")
    case value1[4], value1[5], value1[6]:
        fmt.Println("4 or 5 or 6")
}

value2 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value2[4] {
    case 0, 1:
        fmt.Println("0 or 1")
    case 2, 3:
        fmt.Println("2 or 3")
    case 4, 5, 6:
        fmt.Println("4 or 5 or 6")
}
```

只要 switch 表达式的结果值与某个 case 表达式中的任意一个子表达式的结果值相等，该 case 表达式所属的 case 子句就会被选中。一旦某个 case 子句被选中，其中的附带在 case 表达式后边的那些语句就会被执行。与此同时，其他的所有 case 子句都会被忽略。

如果被选中的 case 子句附带的语句列表中包含了 fallthrough 语句，那么紧挨在它下边的那个 case 子句附带的语句也会被执行。

### switch 表达式不转型

在 Go 语言中，只有类型相同的值之间才有可能被允许进行判等操作。

无类型的常量，会被自动地转换为此种常量的默认类型的值，比如整数 4 的默认类型是 int，又比如浮点数 3.14 的默认类型是 float64。

### case 表达式转型

如果 case 表达式中子表达式的结果值是无类型的常量，那么它的类型会被自动地转换为 switch 表达式的结果类型。

switch 语句会进行有限的类型转换，但肯定不能保证这种转换可以统一它们的类型。还要注意，如果这些表达式的结果类型有某个接口类型，那么一定要小心检查它们的动态值是否都具有可比性（或者说是否允许判等操作）。

## 问题 2：switch 语句对它的 case 表达式有哪些约束？

```golang
// error duplicate case 1 2 4
value := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value[4] {
    case 0, 1, 2, 1:
        fmt.Println("0 or 1 or 2")
    case 2, 3, 4:
        fmt.Println("2 or 3 or 4")
    case 4, 5, 6:
        fmt.Println("4 or 5 or 6")
}
```

switch 语句不允许 case 表达式中的子表达式结果值存在相等的情况，不论这些结果值相等的子表达式，是否存在于不同的 case 表达式中。

上述约束只针对结果值为常量的子表达式。可以通过索引表达式绕过约束。

```golang
value2 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value2[4] {
    case value2[0], value2[1], value2[2]:
        fmt.Println("0 or 1")
    case value2[2], value2[3], value2[4]:
        fmt.Println("2 or 3")
    case value2[4], value2[5], value2[6]:
        fmt.Println("4 or 5 or 6")
}
```


```golang
// error duplicate case byte
// byte 类型是 uint8 类型的别名类型。
value3 := interface{}(byte(127))
switch t := value3.(type) {
    case uint8, uint16:
        fmt.Println("uint8 or uint16")
    case byte:
        fmt.Println("byte")
    default:
        fmt.Println("unsupported type: %T", t)
}
```

类型 switch 语句中的 case 表达式的子表达式，都必须直接由类型字面量表示，而无法通过间接的方式表示。
