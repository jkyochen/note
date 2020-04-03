# pprof trace

`runtime/pprof`
`net/http/pprof`
`runtime/trace`

另外，runtime 代码包中还包含了一些更底层的 API。它们可以被用来收集或输出 Go 程序运行过程中的一些关键指标，并帮助我们生成相应的概要文件以供后续分析时使用。

至于标准工具，主要有 `go tool pprof` 和 `go tool trace` 这两个。它们可以解析概要文件中的信息，并以人类易读的方式把这些信息展示出来。

## CPU 概要文件（CPU Profile）、内存概要文件（Mem Profile）和阻塞概要文件（Block Profile）。

这些概要文件中包含的都是：在某一段时间内，对 Go 程序的相关指标进行多次采样后得到的概要信息。

    + CPU 概要文件，其中的每一段独立的概要信息都记录着，在进行某一次采样的那个时刻，CPU 上正在执行的 Go 代码。

    + 内存概要文件，其中的每一段概要信息都记载着，在某个采样时刻，正在执行的 Go 代码以及堆内存的使用情况，这里包含已分配和已释放的字节数量和对象数量。

    + 阻塞概要文件，其中的每一段概要信息，都代表着 Go 程序中的一个 goroutine 阻塞事件。


`go tool pprof cpuprofile.out` 进入一个基于命令行的交互式界面，并对指定的概要文件进行查阅。

在默认情况下，这些概要文件中的信息并不是普通的文本，它们都是以二进制的形式展现的。protocol buffers 生成的二进制数据流，或者说字节流。

protocol buffers 是一种数据序列化协议，同时也是一个序列化工具。它可以把一个值，比如一个结构体或者一个字典，转换成一段字节流。定义和实现了一种“可以让数据在结构形态和扁平形态之间互相转换”的方式。

    优势有在序列化数据的同时对数据进行压缩，所以它生成的字节流，通常都要比相同数据的其他格式（例如 XML 和 JSON）占用的空间明显小很多。

    既能让我们自己去定义数据序列化和结构化的格式，也允许我们在保证向后兼容的前提下去更新这种格式。


## 怎样让程序对 CPU 概要信息进行采样？

这需要用到 runtime/pprof 包中的 API。更具体地说，在我们想让程序开始对 CPU 概要信息进行采样的时候，需要调用这个代码包中的 StartCPUProfile 函数，而在停止采样的时候则需要调用该包中的 StopCPUProfile 函数。

runtime/pprof.StartCPUProfile 函数（以下简称 StartCPUProfile 函数）在被调用的时候，先会去设定 CPU 概要信息的采样频率，并会在单独的 goroutine 中进行 CPU 概要信息的收集和输出。

注意， StartCPUProfile 函数设定的采样频率总是固定的，即：100 赫兹。也就是说，每秒采样 100 次，或者说每 10 毫秒采样一次。

CPU 的主频指的是，CPU 内核工作的时钟频率，也常被称为 CPU clock speed。这个时钟频率的倒数即为时钟周期（clock cycle），也就是一个 CPU 内核执行一条运算指令所需的时间，单位是秒。

StartCPUProfile 函数设定的 CPU 概要信息采样频率，相对于现代的 CPU 主频来说是非常低的。这主要有两个方面的原因。

一方面，过高的采样频率会对 Go 程序的运行效率造成很明显的负面影响。因此，runtime 包中 SetCPUProfileRate 函数在被调用的时候，会保证采样频率不超过 1MHz（兆赫），也就是说，它只允许每 1 微秒最多采样一次。StartCPUProfile 函数正是通过调用这个函数来设定 CPU 概要信息的采样频率的。

另一方面，经过大量的实验，Go 语言团队发现 100Hz 是一个比较合适的设定。因为这样做既可以得到足够多、足够有用的概要信息，又不至于让程序的运行出现停滞。另外，操作系统对高频采样的处理能力也是有限的，一般情况下，超过 500Hz 就很可能得不到及时的响应了。

在 StartCPUProfile 函数执行之后，一个新启用的 goroutine 将会负责执行 CPU 概要信息的收集和输出，直到 runtime/pprof 包中的 StopCPUProfile 函数被成功调用。

StopCPUProfile 函数也会调用 runtime.SetCPUProfileRate 函数，并把参数值（也就是采样频率）设为 0。这会让针对 CPU 概要信息的采样工作停止。

同时，它也会给负责收集 CPU 概要信息的代码一个“信号”，以告知收集工作也需要停止了。

在接到这样的“信号”之后，那部分程序将会把这段时间内收集到的所有 CPU 概要信息，全部写入到我们在调用 StartCPUProfile 函数的时候指定的写入器中。只有在上述操作全部完成之后，StopCPUProfile 函数才会返回。


## 怎样设定内存概要信息的采样频率？

为 runtime.MemProfileRate 变量赋值即可。

这个变量的含义是，平均每分配多少个字节，就对堆内存的使用情况进行一次采样。如果把该变量的值设为 0，那么，Go 语言运行时系统就会完全停止对内存概要信息的采样。该变量的缺省值是 512 KB，也就是 512 千字节。

注意，如果你要设定这个采样频率，那么越早设定越好，并且只应该设定一次，否则就可能会对 Go 语言运行时系统的采样工作，造成不良影响。比如，只在 main 函数的开始处设定一次。

当我们想获取内存概要信息的时候，还需要调用 runtime/pprof 包中的 WriteHeapProfile 函数。该函数会把收集好的内存概要信息，写到我们指定的写入器中。


## 怎样获取到阻塞概要信息？

我们调用 runtime 包中的 SetBlockProfileRate 函数，即可对阻塞概要信息的采样频率进行设定。该函数有一个名叫 rate 的参数，它是 int 类型的。

这个参数的含义是，只要发现一个阻塞事件的持续时间达到了多少个纳秒，就可以对其进行采样。如果这个参数的值小于或等于 0，那么就意味着 Go 语言运行时系统将会完全停止对阻塞概要信息的采样。

当我们需要获取阻塞概要信息的时候，需要先调用 runtime/pprof 包中的 Lookup 函数并传入参数值 "block"，从而得到一个 `*runtime/pprof.Profile` 类型的值（以下简称 Profile 值）。在这之后，我们还需要调用这个 Profile 值的 WriteTo 方法，以驱使它把概要信息写进我们指定的写入器中。

### debug

debug 参数主要的可选值有两个，即：0 和 1。当 debug 的值为 0 时，通过 WriteTo 方法写进写入器的概要信息仅会包含 go tool pprof 工具所需的内存地址，这些内存地址会以十六进制的形式展现出来。

当该值为 1 时，相应的包名、函数名、源码文件路径、代码行号等信息就都会作为注释被加入进去。另外，debug 为 0 时的概要信息，会经由 protocol buffers 转换为字节流。而在 debug 为 1 的时候，WriteTo 方法输出的这些概要信息就是我们可以读懂的普通文本了。

除此之外，debug 的值也可以是 2。这时，被输出的概要信息也会是普通的文本，并且通常会包含更多的细节。


## runtime/pprof.Lookup 函数的正确调用方式是什么？

runtime/pprof.Lookup 函数（以下简称 Lookup 函数）的功能是，提供与给定的名称相对应的概要信息。这个概要信息会由一个 Profile 值代表。如果该函数返回了一个 nil，那么就说明不存在与给定名称对应的概要信息。

runtime/pprof 包已经为我们预先定义了 6 个概要名称。它们对应的概要信息收集方法和输出方法也都已经准备好了。我们直接拿来使用就可以了。它们是：goroutine、heap、allocs、threadcreate、block 和 mutex。

当我们把"goroutine"传入 Lookup 函数的时候，该函数会利用相应的方法，收集到当前正在使用的所有 goroutine 的堆栈跟踪信息。注意，这样的收集会引起 Go 语言调度器的短暂停顿。

"heap"会使得被输出的内存概要信息默认以“在用空间”（inuse_space）的视角呈现，而"allocs"对应的默认视角则是“已分配空间”（alloc_space）。


## 如何为基于 HTTP 协议的网络服务添加性能分析接口？

```golang
import _ "net/http/pprof"
log.Println(http.ListenAndServe("localhost:8082", nil))
```

一旦 `/debug/pprof/profile` 路径被访问，程序就会去执行对 CPU 概要信息的采样。它接受一个名为 seconds 的查询参数。该参数的含义是，采样工作需要持续多少秒。如果这个参数未被显式地指定，那么采样工作会持续 30 秒。注意，在这个路径下，程序只会响应经 protocol buffers 转换的字节流。我们可以通过 go tool pprof 工具直接读取这样的 HTTP 响应

`go tool pprof http://localhost:6060/debug/pprof/profile?seconds=60`

/debug/pprof/trace。在这个路径下，程序主要会利用 runtime/trace 代码包中的 API 来处理我们的请求。

我们自定义的 HTTP 请求多路复用器 mux 所包含的访问规则与默认的规则很相似，只不过 URL 路径的前缀更短了一些而已。

我们定制 mux 的过程与 net/http/pprof 包中的 init 函数所做的事情也是类似的。这个 init 函数的存在，其实就是我们在前面仅仅导入"net/http/pprof"代码包就能够访问相关路径的原因。
