# Bracket

`{}` 一对不包裹任何东西的花括号，除了可以代表空的代码块之外，还可以用于表示不包含任何内容的数据结构（或者说数据类型）。

`struct{}`，它就代表了不包含任何字段和方法的、空的结构体类型。

空接口 `interface{}` 则代表了不包含任何方法定义的、空的接口类型。

对于一些集合类的数据类型来说，`{}` 还可以用来表示其值不包含任何元素，比如空的切片值 `[]string{}`，以及空的字典值 `map[int]string{}`。

圆括号中 `[]string` 是一个类型字面量。所谓类型字面量，就是用来表示数据类型本身的若干个字符。