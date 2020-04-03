# http

http.Get 函数会在内部使用缺省的 HTTP 客户端，并且调用它的 Get 方法以完成功能。这个缺省的 HTTP 客户端是由 `net/http` 包中的公开变量 DefaultClient 代表的，其类型是 `*http.Client`。它的基本类型也是可以被拿来使用的，甚至它还是开箱即用的。

## http.Client 类型中的 Transport 字段代表着什么？

http.Client 类型中的 Transport 字段代表着：向网络服务发送 HTTP 请求，并从网络服务接收 HTTP 响应的操作过程。也就是说，该字段的方法 RoundTrip 应该实现单次 HTTP 事务（或者说基于 HTTP 协议的单次交互）需要的所有步骤。

这个字段是 http.RoundTripper 接口类型的，它有一个由 http.DefaultTransport 变量代表的缺省值（以下简称 DefaultTransport）。当我们在初始化一个 http.Client 类型的值（以下简称 Client 值）的时候，如果没有显式地为该字段赋值，那么这个 Client 值就会直接使用 DefaultTransport。

顺便说一下，http.Client 类型的 Timeout 字段，代表的正是前面所说的单次 HTTP 事务的超时时间，它是 time.Duration 类型的。它的零值是可用的，用于表示没有设置超时时间。

http.Transport 类型还包含了很多其他的字段，其中有一些字段是关于操作超时的。

-   IdleConnTimeout：含义是空闲的连接在多久之后就应该被关闭。DefaultTransport 会把该字段的值设定为 90 秒。如果该值为 0，那么就表示不关闭空闲的连接。注意，这样很可能会造成资源的泄露。
-   ResponseHeaderTimeout：含义是，从客户端把请求完全递交给操作系统到从操作系统那里接收到响应报文头的最大时长。DefaultTransport 并没有设定该字段的值。
-   ExpectContinueTimeout：含义是，在客户端递交了请求报文头之后，等待接收第一个响应报文头的最长时间。在客户端想要使用 HTTP 的“POST”方法把一个很大的报文体发送给服务端的时候，它可以先通过发送一个包含了“Expect: 100-continue”的请求报文头，来询问服务端是否愿意接收这个大报文体。这个字段就是用于设定在这种情况下的超时时间的。注意，如果该字段的值不大于 0，那么无论多大的请求报文体都将会被立即发送出去。这样可能会造成网络资源的浪费。DefaultTransport 把该字段的值设定为了 1 秒。
-   TLSHandshakeTimeout：TLS 是 Transport Layer Security 的缩写，可以被翻译为传输层安全。这个字段代表了基于 TLS 协议的连接在被建立时的握手阶段的超时时间。若该值为 0，则表示对这个时间不设限。DefaultTransport 把该字段的值设定为了 10 秒。

## 问题：http.Server 类型的 ListenAndServe 方法都做了哪些事情？

http.Server 类型的 ListenAndServe 方法的功能是：监听一个基于 TCP 协议的网络地址，并对接收到的 HTTP 请求进行处理。这个方法会默认开启针对网络连接的存活探测机制，以保证连接是持久的。同时，该方法会一直执行，直到有严重的错误发生或者被外界关掉。当被外界关掉时，它会返回一个由 http.ErrServerClosed 变量代表的错误值。

ListenAndServe 方法主要会做下面这几件事情。

1. 检查当前的 http.Server 类型的值（以下简称当前值）的 Addr 字段。该字段的值代表了当前的网络服务需要使用的网络地址，即：IP 地址和端口号。如果这个字段的值为空字符串，那么就用":http"代替。也就是说，使用任何可以代表本机的域名和 IP 地址，并且端口号为 80。
2. 通过调用 net.Listen 函数在已确定的网络地址上启动基于 TCP 协议的监听。
3. 检查 net.Listen 函数返回的错误值。如果该错误值不为 nil，那么就直接返回该值。否则，通过调用当前值的 Serve 方法准备接受和处理将要到来的 HTTP 请求。

### net.Listen 函数都做了哪些事情

1. 解析参数值中包含的网络地址隐含的 IP 地址和端口号；
2. 根据给定的网络协议，确定监听的方法，并开始进行监听。

### http.Server 类型的 Serve 方法是怎样接受和处理 HTTP 请求的

在一个 for 循环中，网络监听器的 Accept 方法会被不断地调用，该方法会返回两个结果值；第一个结果值是 net.Conn 类型的，它会代表包含了新到来的 HTTP 请求的网络连接；第二个结果值是代表了可能发生的错误的 error 类型值。

如果这个错误值不为 nil，除非它代表了一个暂时性的错误，否则循环都会被终止。如果是暂时性的错误，那么循环的下一次迭代将会在一段时间之后开始执行。

如果这里的 Accept 方法没有返回非 nil 的错误值，那么这里的程序将会先把它的第一个结果值包装成一个 `*http.conn` 类型的值（以下简称 conn 值），然后通过在新的 goroutine 中调用这个 conn 值的 serve 方法，来对当前的 HTTP 请求进行处理。
