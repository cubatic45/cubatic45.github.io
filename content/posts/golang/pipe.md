+++
title = 'Pipe'
date = 2024-07-30T15:46:00+08:00
draft = false
tags = ['golang']
+++


## Pipe

io.pipe 结构

```go
type pipe struct {
    wrMu sync.Mutex   // 序列化写操作的互斥锁
    wrCh chan []byte  // 用于从写者向读者发送数据的通道
    rdCh chan int     // 用于从读者向写者发送读取字节数的通道

    once sync.Once    // 确保'done'通道只关闭一次
    done chan struct{}// 用于指示管道关闭的通道
    rerr onceError    // 存储关闭时发生的读错误
    werr onceError    // 存储关闭时发生的写错误
}
```

 - 第一个select语句检查管道是否已关闭，通过尝试读取p.done通道。如果管道已关闭，则返回错误。
 - 第二个select语句等待从写者通过p.wrCh发送的数据或管道关闭。
    - 如果收到数据，将数据复制到b，并通过p.rdCh将读取的字节数发送回写者。
    - 如果管道没有关闭，则会阻塞，直到管道关闭。
    - 如果管道关闭，则返回错误。

```go
func (p *pipe) read(b []byte) (n int, err error) {
	select {
	case <-p.done:
		return 0, p.readCloseError()
	default:
	}

	select {
	case bw := <-p.wrCh:
		nr := copy(b, bw)
		p.rdCh <- nr
		return nr, nil
	case <-p.done:
		return 0, p.readCloseError()
	}
}
```

 - 第一个select语句检查管道是否已关闭。如果已关闭，则返回写错误。
 - p.wrMu.Lock()确保只有一个写操作可以进行。
 - for循环持续进行直到所有数据b都被写入。
    - 尝试通过p.wrCh发送数据b。
    - 等待读者通过p.rdCh发送的读取字节数，然后调整b以移除已写入的字节。
    - 如果在此过程中管道关闭，则返回写错误。

```go
func (p *pipe) write(b []byte) (n int, err error) {
	select {
	case <-p.done:
		return 0, p.writeCloseError()
	default:
		p.wrMu.Lock()
		defer p.wrMu.Unlock()
	}

	for once := true; once || len(b) > 0; once = false {
		select {
		case p.wrCh <- b:
			nw := <-p.rdCh
			b = b[nw:]
			n += nw
		case <-p.done:
			return n, p.writeCloseError()
		}
	}
	return n, nil
}
```

## example

```go
package main

import (
	"fmt"
	"io"
)

func main() {
	pr, pw := io.Pipe()
	data := []byte("hello")
	go func() {
		pw.Write(data)
		pw.Close()
	}()
	for range 5 {
		rdata := make([]byte, 2)
		pr.Read(rdata)
		fmt.Println(string(rdata))
	}
}
```

output:

```shell
he
ll
o
 
 
```

write 第一次写入 hello，第二次写入 ll，第三次写入 o。

read 第一次读取 he，第二次读取 ll，第三次读取 o，第四次读取空，第五次读取空。
