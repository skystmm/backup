---
title: go学习笔记(四)——并发
date: 2020-03-18 20:26:14
tags: [go]
categories: go
---


# 1. goroutine
## 什么是goroutine
`goroutine`是golang的最小执行单元，每个go程序至少会有一个主goroutine，这里可以类比成java中额主线程。为了更好的理解goroutine，可以将进程，线程和goroutine(其他语言中的协程)进行类比。

<!-- more -->
## 进程，线程和goroutine
![进程，线程和协程关系](http://image.stxl117.top/ptcg.jpg)

**进程**是操作系统进行资源分配，调度和执行的基本单元。当操作系统分配一个进程时，会生成一个唯一对应的PCB块以及为该进程分配专有的系统资源。后续操作系统的调度会通过对应的PCB块来进行。

**线程**是程序执行流的最小单元，一个标准的线程由线程ID，当前指令指针(PC），寄存器集合和堆栈组成。一般情况下进程与线程为1:n的关系，同一个进程生成的线程会共享改进程的堆上资源。通常情况线程的调度由操作系统内核态进行调度。

**协程(goroutine)**是更轻量级的线程，或者可以理解为用户态的线程。协程的调度实现答题可以分为两种，一种是用户态的调度实现，比如python中有名的gevent，就是通过用户态调度器实现协程间的调度，这种调度方式存在的一个弊端是由于调度完全是在用户态进行的，所以内核线程与协程的映射关系为1:N,即一个线程内的协程都是交替执行，不能并发；而golang则是在此基础上进行了优化，采用两级调度模型，实现了协程的并发，所以goroutine和内核线程的对应关系为n:m。


>1.参考文章2中对于两级调度并发有更进一步的讲解
2.随着这两年互联网的发展，越来越多的语言开始对协程进行了支持，从最开始Python2的greenlet,gevent(线程和协程的对应关系为1:N,用户级线程模型)到Python3在语言层面进行了支持。java中也有kilim和Quasar进行了支持。

# 2. go关键字
`go`关键字是golang提供的生成和使用goroutine的关键字，如下代码
```go
  func test(){
    fmt.Println("new goroutine")
  }
  go test()
```
`test`函数将在一个新建的goroutine中执行。由于golang在语言层面对于goroutine进行了支持，所以不用想在java中通过实现`Runnable`接口或者继承`Thread`后，还要显示的调用`start`方法才可以在新线程中执行相应的功能。同时由于goroutine的上下文切换开销和所需的内存空间更小(2k)，相同性能的机器可以支撑的goroutine数量远远多于可以支持的线程数量。

# 3. chan关键字
golang将CSP模型作为其并发的基础。正如golang著名的口号一样：*"不要以共享内存的方式来通信，相反，要通过通信来共享内存"*。既然要用通信的方式来共享内存，所以go就有了channel的出现，来支持已通信的方式共享内存。
```go
    channel = make(chan int ,n)
    channel <- 1 //将信息写入channel
    i := <-channel //从channel中读取信息
```
对于channel如何为golang的并发保驾护航，将在后面进行详细描述。


# 4. select关键字
第一次看到`select`这个关键字的时候，第一反应就是经典的select模型。对想的没错，golang中将通信过程中的通过`select`进行了语言层面支持了通信层面的多路复用器。
```go
  select{
      case <-ch:
      //TODO
      case xxx:
      //TODO
  }
```
可以看到golang中对于`select`的使用和前文中提到的`swtich`的关键字使用方法类似。

# 5. go的内存模型

类似于java中的JMM,go也通过定制内存模型，保证并发的正确性。类比于JMM中的happens-before原则，go中的happens-before原则如下，使用a`->`b标识a操作happens-before b操作。大体上go的内存模型可以分为以下几类

## 初始化
1.被引入包的init函数优先于本包的所有方法
```go
package p
import q
//q.init -> p.*
```
2.导入的所有包的init函数->main函数的执行

## goroutine相关

3.goroutine的创建 -> 其执行

4.goroutine无法确保在程序中的任何事件发生之前退出
```go
func main(){
  go test()
  fmt.Println("mian func")
}

func test(){
  fmt.Println("test")
}
//test不一定能输出
```

## channel管道
5.一个goroutine向一个channel发送数据 ->另一个goroutine从本channel中接收数据

6.当channel执行关闭操作后，channel中已有数据仍可被获取，获取完后，在进行获取则为零值。
```go
var ch = make(chan int ,5)
func main(){

	for i:=0;i<5;i++{
		ch <- i
	}
	close(ch)
	for i:=0;i<10;i++{
		fmt.Println(<-ch)
	}
}
//结果 0 1 2 3 4 0 0 0 0 0
```
7.无缓存的channel的获取数据会阻塞到向channel中发送数据
```go
var ch = make(chan struct{} ,0)
func main(){
	go func(){
		fmt.Println("step one")
		ch<- struct{}{}
	}()
	<-ch
	fmt.Println("step two")
}
```
> 可以用无缓存的channel来实现锁机制

8.对于一个容量为N的channel第k次接收数据 ->该channel的第k+n次发送
>保证被消费，channel满了后会阻塞，直到有被消费调的信息，类比java中的BlockingQuene

## 锁

>又见CAS
`sync`包提供了两种锁 :`sync.Mutex`和`sync.WRMutex`
 `sync.Mutex`独占锁
 `sync.WRMutex`读写锁
类比Java中相应lock实现,区别go中锁不可重入

9.对于任何 sync.Mutex 或 sync.RWMutex 类型的变量 l 以及 n < m ，对 l.Unlock() 的第 n 次调用在对 l.Lock() 的第 m 次调用返回前发生。

10.对于任何 sync.RWMutex 类型的变量 l 对 l.RLock 的调用，存在一个这样的 n，使得 l.RLock 在对 l.Unlock 的第 n 次调用之后发生（返回），且与其相匹配的 l.RUnlock 在对 l.Lock的第 n+1 次调用之前发生。


## Once类型——单例好帮手

11.通过 once.Do(f) 对 f() 的单次调用在对任何其它的 once.Do(f) 调用返回之前发生（返回）。

>1.懒汉式在go中的最佳实践，思路同java中的线程安全的懒汉式单例
2.sync 包通过 Once 类型为存在多个Go程的初始化提供了安全的机制。 多个线程可为特定的 f 执行 once.Do(f)，但只有一个会运行 f()，而其它调用会一直阻塞，直到 f() 返回。


# goroutine和channel随想
1.[ 示例: 并发的非阻塞缓存](http://shouce.jb51.net/gopl-zh/ch9/ch9-07.html)
2.java中可以通过ForkJoinPool来实现两级线程模型自定义调度器来实现可并发的java协程？

> e.g.1目录文件遍历和空间统计

>参考内容：
> [go内存模型-en](https://golang.org/ref/mem)
> [go内存模型-zh](https://go-zh.org/ref/mem)
> [Goroutine并发调度模型深度解析&手撸一个协程池](https://juejin.im/entry/5b2878c7f265da5977596ae2)
> [深入理解golang之channel](https://juejin.im/post/5decff136fb9a016544bce67)
