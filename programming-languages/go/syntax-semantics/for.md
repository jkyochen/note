# for

## for...range

```golang
number1 := []int{1, 2, 3, 4, 5, 6}
for i := range number1 {
    if i == 3 {
        // |= 位或
        number1[i] |= i
    }
}
fmt.Println(number1)

number2 := [...]int{1, 2, 3, 4, 5, 6}
maxIndex2 := len(number2) - 1
for i, e := range number2 {
    if i == maxIndex2 {
        number2[0] += e
    } else {
        number2[i + 1] += e
    }
}
fmt.Println(number2)

number3 := []int{1, 2, 3, 4, 5, 6}
maxIndex3 := len(number3) - 1
for i, e := range number3 {
    if i == maxIndex3 {
        number3[0] += e
    } else {
        number3[i + 1] += e
    }
}
fmt.Println(number3)
```

range 表达式的结果值可以是数组、数组的指针、切片、字符串、字典或者允许接收操作的通道中的某一个，并且结果值只能有一个。

一个切片，那么迭代变量就可以有两个，右边的迭代变量代表当次迭代对应的某一个元素值，而左边的迭代变量则代表该元素值在切片中的索引值。


可以忽略迭代时的下标。

```golang
var times [5][0]int
for range times {
    fmt.Println("hello")
}
```

### 注意

1. range 表达式只会在 for 语句开始执行时被求值一次，无论后边会有多少次迭代。
2. range 表达式的求值结果会被复制，也就是说，被迭代的对象是 range 表达式结果值的副本而不是原值。

### channel

可以检测到通道关闭。
