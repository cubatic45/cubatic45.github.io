+++
title = 'Go Channel'
date = 2025-07-09T09:10:33+08:00
draft = false
tags = ['golang']
+++

以下代码为 golang 1.24.5

## chan

### chan 底层 结构

```go
const (
	maxAlign  = 8 // 最大对齐字节数（通常与架构相关）
	hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1)) // hchan 结构体的大小（含对齐填充）
	debugChan = false // 调试标志，是否开启 channel 调试
)

// Go 语言运行时中 channel 的底层结构
type hchan struct {
	qcount   uint           // 队列中当前的元素数量
	dataqsiz uint           // 环形缓冲区的大小（也就是 buf 中元素的总数）
	buf      unsafe.Pointer // 指向实际数据缓冲区（数组，元素个数为 dataqsiz）
	elemsize uint16         // 每个元素的大小（字节）
	synctest bool           // 是否用于 sync 包的测试中（true 表示在 synctest 模式中创建）
	closed   uint32         // 标志位，表示 channel 是否已关闭
	timer    *timer         // 用于超时操作的定时器指针（如 select 的超时）
	elemtype *_type         // 指向元素类型的类型描述符
	sendx    uint           // 当前发送的索引（用于环形缓冲区）
	recvx    uint           // 当前接收的索引（用于环形缓冲区）
	recvq    waitq          // 等待接收的 goroutine 队列（recv 阻塞队列）
	sendq    waitq          // 等待发送的 goroutine 队列（send 阻塞队列）

	// lock 保护 hchan 中的所有字段，
	// 以及阻塞在该 channel 上的 sudog 的多个字段。
	// ⚠️ 在持有该锁时不要修改其他 G 的状态（尤其是不要唤醒 G），
	// 否则可能与栈收缩操作发生死锁。
	lock mutex
}

// 等待队列结构，用于接收和发送阻塞的 goroutine 链表
type waitq struct {
	first *sudog // 队首 goroutine
	last  *sudog // 队尾 goroutine
}
```

先来解释 `maxAlign  = 8` 和 `hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))`

这两句的意思是 将 hchanSize 向上对齐为 8 的整数倍

---

```
hchanSize = size + (-size & (align - 1))
size = 3
align = 8

8 - 1 = 7
=> 二进制: 0000 0111

-3 的二进制表示（假设是 int32）是：
  原码：0000 0011 （3）
  反码：1111 1100
  补码：1111 1101 （即 -3）

所以 -3 的补码是：1111 1101


-3: 1111 1101
 7: 0000 0111
--------------
    0000 0101 => 5

3+5=8
```

---

### 初始化 chan

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem

	// compiler checks this but be safe.
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0: // 无缓存的情况
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case !elem.Pointers(): // chan 类型是指针
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true)) // 申请 连续内存
		c.buf = add(unsafe.Pointer(c), hchanSize) // 将缓存区指针指向缓存区
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true) // 单独分配缓存区内存
	}

	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	if getg().syncGroup != nil {
		c.synctest = true
	}
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c
}
```

---

### 写 chan

```go
// entry point for c <- x from compiled code.
//
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, sys.GetCallerPC())
}

/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2) // 向 nil chan 写数据会阻塞当前 goroutine
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(chansend))
	}

	if c.synctest && getg().syncGroup == nil {
		panic(plainError("send on synctest channel from outside bubble"))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
	if !block && c.closed == 0 && full(c) { // golang 对单字（word-sized）的 observation 是原子操作
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock) // 加锁

	if c.closed != 0 { // 向已经关闭的 chan 写会 panic
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil { // 如果找到一个只在等待的 g，直接将数据给 g 而不是将数据存到buf
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3) // send 函数除开会复制数据还会调用 goready
		return true
	}

	if c.qcount < c.dataqsiz { // buf 未满
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx) // 获取缓存区中待写入的指针
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep) // 写入数据
		c.sendx++ // 带写入指针++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block { // 非阻塞直接返回失败
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg) // 进发送队列等待
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	reason := waitReasonChanSend
	if c.synctest {
		reason = waitReasonSynctestChanSend
	}
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanSend, 2) // 阻塞当前 goroutine，由于 chan 的锁没有释放，下一个 goroutine 写数据也会阻塞到获取锁的位置
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

#### 总结一下写 chan

---

- 阻塞写 nil chan 会阻塞

```go
package main

func main() {
	var ch chan int
	ch <- 1
}
```

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
	/tmp/sandbox1677365408/prog.go:5 +0x1c
```

---

- 向已经关闭的 chan 写会 panic

---

```go
package main

func main() {
	ch := make(chan int)
	close(ch)
	ch <- 1
}
```

```bash
panic: send on closed channel

goroutine 1 [running]:
main.main()
	/tmp/sandbox2376909824/prog.go:6 +0x37
```

---

- 有等待读者，写 chan 会直接写入等待读者，否则写入 buf

- 阻塞写 满 chan 会阻塞，并且后续 goroutine 也会阻塞（如果其中未有读者）

### 读 chan

```go
// entry points for <- c from compiled code.
//
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2) // 阻塞读 nil chan 会阻塞
		throw("unreachable")
	}

	if c.synctest && getg().syncGroup == nil {
		panic(plainError("receive on synctest channel from outside bubble"))
	}

	if c.timer != nil {
		c.timer.maybeRunChan()
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) {
		// After observing that the channel is not ready for receiving, we observe whether the
		// channel is closed.
		//
		// Reordering of these checks could lead to incorrect behavior when racing with a close.
		// For example, if the channel was open and not empty, was closed, and then drained,
		// reordered reads could incorrectly indicate "open and empty". To prevent reordering,
		// we use atomic loads for both checks, and rely on emptying and closing to happen in
		// separate critical sections under the same lock.  This assumption fails when closing
		// an unbuffered channel with a blocked send, but that is an error condition anyway.
		if atomic.Load(&c.closed) == 0 {
			// Because a channel cannot be reopened, the later observation of the channel
			// being not closed implies that it was also not closed at the moment of the
			// first observation. We behave as if we observed the channel at that moment
			// and report that the receive cannot proceed.
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock) // 加锁

	if c.closed != 0 {
		if c.qcount == 0 { // chan 已经关闭，而且缓存区没有数据
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
		// The channel has been closed, but the channel's buffer have data.
	} else {
		// Just found waiting sender with not closed.
		if sg := c.sendq.dequeue(); sg != nil { // 如果发送队列有 goroutine，直接接收数据
			// Found a waiting sender. If buffer is size 0, receive value
			// directly from sender. Otherwise, receive from head of queue
			// and add sender's value to the tail of the queue (both map to
			// the same buffer slot because the queue is full).
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

	if c.qcount > 0 { // 缓存区有数据
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block { // 非阻塞直接返回
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg

	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg) // 当前 goroutine 进接收等待队列
	if c.timer != nil {
		blockTimerChan(c)
	}

	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	reason := waitReasonChanReceive
	if c.synctest {
		reason = waitReasonSynctestChanReceive
	}
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanRecv, 2) // 阻塞当前 goroutine

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	if c.timer != nil {
		unblockTimerChan(c)
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg) // 释放 mysg 到 pool
	return true, success
}

```

#### 总结一下读 chan

---

- 阻塞读 nil chan 会阻塞

```go
package main

func main() {
	var ch chan int
	<-ch
}
```

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive (nil chan)]:
main.main()
	/tmp/sandbox565923349/prog.go:5 +0x17
```

---

- 向已经关闭的 chan 读不会 panic，并且能正常读

---

```go
package main

func main() {
	ch := make(chan int)
	go func() { ch <- 1; close(ch) }()
	print(<-ch)
}
```

```bash
1
```

---

- 有等待写者，读 chan 会直接读取等待写者，否则读 buf

- 阻塞读 空 chan 会阻塞，并且后续 goroutine 也会阻塞（如果其中未有写者）

