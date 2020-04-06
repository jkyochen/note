# Struct

所有继承来的方法的接收者参数依然是那个匿名成员本身，而不是当前的变量。

```golang
type Cache struct {
    m map[string]string
    sync.Mutex
}

func (p *Cache) Lookup(key string) string {
    p.Lock()
    defer p.Unlock()

    return p.m[key]
}
```

Cache 结构体类型通过嵌入一个匿名的 sync.Mutex 来继承它的 Lock 和 Unlock 方法. 但是在调用 p.Lock()和 p.Unlock()时, p 并不是 Lock 和 Unlock 方法的真正接收者, 而是会将它们展开为 p.Mutex.Lock()和 p.Mutex.Unlock()调用. 这种展开是编译期完成的, 并没有运行时代价。

```golang
type AnimalCategory struct {
    kingdom string
    phylum string
    class string
    order string
    family string
    genus string
    species string
}

func (ac AnimalCategory) String() string {
    return fmt.Sprintf("%s%s%s%s%s%s%s", ac.kingdom, ac.phylum, ac.class, ac.order, ac.family, ac.genus, ac.species)
}

func main() {
    category := AnimalCategory{species: "cat"}
    fmt.Printf("The animal category: %s\n", category)
}
```

函数是独立的程序实体。我们可以声明有名字的函数，也可以声明没名字的函数，还可以把它们当做普通的值传来传去。我们能把具有相同签名的函数抽象成独立的函数类型，以作为一组输入、输出（或者说一类逻辑组件）的代表。

方法却不同，它需要有名字，不能被当作值来看待，最重要的是，它必须隶属于某一个类型。方法所属的类型会通过其声明中的接收者（receiver）声明体现出来。

接收者声明就是在关键字 func 和方法名称之间的圆括号包裹起来的内容，其中必须包含确切的名称和类型字面量。

接收者的类型其实就是当前方法所属的类型，而接收者的名称，则用于在当前方法中引用它所属的类型的当前值。

通过该方法的接收者名称，我们可以在其中引用到当前值的任何一个字段，或者调用到当前值的任何一个方法（也包括 String 方法自己）。

## String

在 Go 语言中，我们可以通过为一个类型编写名为 String 的方法，来自定义该类型的字符串表示形式。这个 String 方法不需要任何参数声明，但需要有一个 string 类型的结果声明。

## Type

方法隶属的类型其实并不局限于结构体类型，但必须是某个自定义的数据类型，并且不能是任何接口类型。

一个数据类型关联的所有方法，共同组成了该类型的方法集合。同一个方法集合中的方法不能出现重名。并且，如果它们所属的是一个结构体类型，那么它们的名称与该类型中任何字段的名称也不能重复。

## 嵌入字段 / 匿名字段

```golang

type AnimalCategory struct {
    kingdom string
    phylum string
    class string
    order string
    family string
    genus string
    species string
}

func (ac AnimalCategory) String() string {
    return fmt.Sprintf("%s%s%s%s%s%s%s", ac.kingdom, ac.phylum, ac.class, ac.order, ac.family, ac.genus, ac.species)
}

type Animal struct {
    scientificName string
    AnimalCategory
}

func (a Animal) Category() string {
    return a.AnimalCategory.String()
}

func (a Animal) String() string {
    return fmt.Sprintf("%s (category: %s)", a.scientificName, a.AnimalCategory)
}

type Cat struct {
    name string
    Animal
}

func (cat Cat) String() string {
    return fmt.Sprintf("%s (category: %s, name: %q)", cat.scientificName, cat.Animal.AnimalCategory, cat.name)
}

func (cat *Cat) SetName(name string) {
    cat.name = name
}

func main() {
    category := AnimalCategory{species: "cat"}
    animal := Animal{
        scientificName: "Americn Shorthair",
        AnimalCategory: category,
    }
    fmt.Printf("The animal: %s\n", animal)
}
```

字段声明 AnimalCategory 代表了 Animal 类型的一个嵌入字段。

Go 语言规范规定，如果一个字段的声明中只有字段的类型名而没有字段的名称，那么它就是一个嵌入字段，也可以被称为匿名字段。

我们可以通过此类型变量的名称后跟“.”，再后跟嵌入字段类型的方式引用到该字段。也就是说，嵌入字段的类型既是类型也是名称。

在某个代表变量的标识符的右边加“.”，再加上字段名或方法名的表达式被称为选择表达式，它用来表示选择了该变量的某个字段或者方法。

嵌入字段的方法集合会被无条件地合并进被嵌入类型的方法集合中。

### 屏蔽

在两个同名的成员一个是字段，另一个是方法的情况下，这种“屏蔽”现象依然会存在。

### 多个嵌入字段

如果处于同一个层级的多个嵌入字段拥有同名的字段或方法，那么从被嵌入类型的值那里，选择此名称的时候就会引发一个编译错误，因为编译器无法确定被选择的成员到底是哪一个。

## 问题 1：Go 语言是用嵌入字段实现了继承吗？

类型之间的组合采用的是非声明的方式，我们不需要显式地声明某个类型实现了某个接口，或者一个类型继承了另一个类型。

同时，类型组合也是非侵入式的，它不会破坏类型的封装或加重类型之间的耦合。

## 问题 2：值方法和指针方法都是什么意思，有什么区别？

方法的接收者类型必须是某个自定义的数据类型，而且不能是接口类型或接口的指针类型。

所谓的值方法，就是接收者类型是非指针的自定义数据类型的方法。

Cat 可以被叫做 `*Cat` 的基本类型。你可以认为这种指针类型的值表示的是指向某个基本类型值的指针。

通过把取值操作符 \* 放在这样一个指针值的左边来组成一个取值表达式，以获取该指针值指向的基本类型值，也可以通过把取址操作符 & 放在一个可寻址的基本类型值的左边来组成一个取址表达式，以获取该基本类型值的指针值。

### 那么值方法和指针方法之间有什么不同点呢？

1. 副本。

    值方法的接收者是该方法所属的那个类型值的一个副本。我们在该方法内对该副本的修改一般都不会体现在原值上，除非这个类型本身是某个引用类型（比如切片或字典）的别名类型。

    指针方法的接收者，是该方法所属的那个基本类型值的指针值的一个副本。我们在这样的方法内对该副本指向的值进行修改，却一定会体现在原值上。

2. 一个自定义数据类型的方法集合中仅会包含它的所有值方法，而该类型的指针类型的方法集合却囊括了前者的所有方法，包括所有值方法和所有指针方法。

    严格来讲，我们在这样的基本类型的值上只能调用到它的值方法。但是，Go 语言会适时地为我们进行自动地转译，使得我们在这样的值上也能调用到它的指针方法。

    比如，在 Cat 类型的变量 cat 之上，之所以我们可以通过 `cat.SetName("monster")` 修改猫的名字，是因为 Go 语言把它自动转译为了 `(&cat).SetName("monster")`，即：先取 cat 的指针值，然后在该指针值上调用 SetName 方法。

3. 一个类型的方法集合中有哪些方法与它能实现哪些接口类型是息息相关的。如果一个基本类型和它的指针类型的方法集合是不同的，那么它们具体实现的接口类型的数量就也会有差异，除非这两个数量都是零。
