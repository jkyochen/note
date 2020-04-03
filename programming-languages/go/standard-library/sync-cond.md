# sync.Cond

条件变量在这里的最大优势就是在效率方面的提升。当共享资源的状态不满足条件的时候，想操作它的线程再也不用循环往复地做检查了，只要等待通知就好了。

```golang
var mailbox uint8
var lock sync.RWMutex
sendCond := sync.NewCond(&lock)
recvCond := sync.NewCond(lock.RLocker())

go func() {

    lock.Lock()
    for mailbox == 1 {
        sendCond.Wait()
    }
    mailbox = 1
    lock.Unlock()
    recvCond.Signal()

}()

go func() {

    lock.RLock()
    for mailbox == 0 {
        recvCond.Wait()
    }
    mailbox = 0
    lock.RUnlock()
    sendCond.Signal()

}()
```

## 条件变量怎样与互斥锁配合使用？

这道题的典型回答是：条件变量的初始化离不开互斥锁，并且它的方法有的也是基于互斥锁的。

条件变量提供的方法有三个：等待通知（wait）、单发通知（signal）和广播通知（broadcast）。

lock 变量的 Lock 方法和 Unlock 方法分别用于对其中写锁的锁定和解锁，它们与 sendCond 变量的含义是对应的。

recvCond 变量代表的是专门为获取情报而准备的条件变量。

利用条件变量可以实现单向的通知，而双向的通知则需要两个条件变量。这也是条件变量的基本使用规则。

条件变量的通知具有即时性。也就是说，如果发送通知的时候没有 goroutine 为此等待，那么该通知就会被直接丢弃。

## 条件变量的 Wait 方法做了什么？

1. 把调用它的 goroutine（也就是当前的 goroutine）加入到当前条件变量的通知队列中。
2. 解锁当前的条件变量基于的那个互斥锁。
3. 让当前的 goroutine 处于等待状态，等到通知到来时再决定是否唤醒它。此时，这个 goroutine 就会阻塞在调用这个 Wait 方法的那行代码上。
4. 如果通知到来并且决定唤醒这个 goroutine，那么就在唤醒它之后重新锁定当前条件变量基于的互斥锁。自此之后，当前的 goroutine 就会继续执行后面的代码了。

## 为什么先要锁定条件变量基于的互斥锁，才能调用它的 Wait 方法？

因为条件变量的 Wait 方法在阻塞当前的 goroutine 之前，会解锁它基于的互斥锁，所以在调用该 Wait 方法之前，我们必须先锁定那个互斥锁，否则在调用这个 Wait 方法时，就会引发一个不可恢复的 panic。

## 为什么要用 for 语句来包裹调用其 Wait 方法的表达式，用 if 语句不行吗？

如果一个 goroutine 因收到通知而被唤醒，但却发现共享资源的状态，依然不符合它的要求，那么就应该再次调用条件变量的 Wait 方法，并继续等待下次通知的到来。

1. 有多个 goroutine 在等待共享资源的同一种状态。比如，它们都在等 mailbox 变量的值不为 0 的时候再把它的值变为 0，这就相当于有多个人在等着我向信箱里放置情报。虽然等待的 goroutine 有多个，但每次成功的 goroutine 却只可能有一个。别忘了，条件变量的 Wait 方法会在当前的 goroutine 醒来后先重新锁定那个互斥锁。在成功的 goroutine 最终解锁互斥锁之后，其他的 goroutine 会先后进入临界区，但它们会发现共享资源的状态依然不是它们想要的。这个时候，for 循环就很有必要了。

2. 共享资源可能有的状态不是两个，而是更多。比如，mailbox 变量的可能值不只有 0 和 1，还有 2、3、4。这种情况下，由于状态在每次改变后的结果只可能有一个，所以，在设计合理的前提下，单一的结果一定不可能满足所有 goroutine 的条件。那些未被满足的 goroutine 显然还需要继续等待和检查。

3. 有一种可能，共享资源的状态只有两个，并且每种状态都只有一个 goroutine 在关注。不过，即使是这样，使用 for 语句仍然是有必要的。原因是，在一些多 CPU 核心的计算机系统中，即使没有收到条件变量的通知，调用其 Wait 方法的 goroutine 也是有可能被唤醒的。这是由计算机硬件层面决定的，即使是操作系统（比如 Linux）本身提供的条件变量也会如此。

## 条件变量的 Signal 方法和 Broadcast 方法有哪些异同？

条件变量的 Wait 方法总会把当前的 goroutine 添加到通知队列的队尾，而它的 Signal 方法总会从通知队列的队首开始，查找可被唤醒的 goroutine。所以，因 Signal 方法的通知，而被唤醒的 goroutine 一般都是最早等待的那一个。

Broadcast 会唤醒所有为此等待的 goroutine。

与 Wait 方法不同，条件变量的 Signal 方法和 Broadcast 方法并不需要在互斥锁的保护下执行。恰恰相反，我们最好在解锁条件变量基于的那个互斥锁之后，再去调用它的这两个方法。这更有利于程序的运行效率。

条件变量的通知具有即时性。也就是说，如果发送通知的时候没有 goroutine 为此等待，那么该通知就会被直接丢弃。
