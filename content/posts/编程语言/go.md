---
title: Golang
date: 2022-06-15T23:21:48+08:00
lastmod: 2022-06-15T23:21:48+08:00
cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/6_15/go.jpg
categories:
  - 编程语言
tags:
  - golang
# nolastmod: true
draft: false
---

总结一下在go语言的学习与实践中学到的知识

<!--more-->

## 一、底层原理

### 1、Go的协程

多进程、多线程已经提高了系统的并发能力，但是在当今互联网高并发场景下，为每个任务都创建一个线程是不现实的，因为会消耗大量的内存(进程虚拟内存会占用4GB[32位操作系统], 而线程也要大约4MB)。

Go中，协程被称为goroutine，它非常轻量，一个goroutine只占几KB，并且这几KB就足够goroutine运行完，这就能在有限的内存空间内支持大量goroutine，支持了更多的并发。虽然一个goroutine的栈只占几KB，但实际是可伸缩的，如果需要更多内容，`runtime`会自动为goroutine分配。

与python和其他语言的协程相比，go的协程有很大不同。

python的协程是通常意义上的协程，两大特征就是**可中断和可恢复**。无法利用多核，不会并行。更适用于IO密集程序。

**从运行机制上来说**，coroutine 的运行机制属于**协作式任务**处理， 程序需要主动交出控制权，宿主才能获得控制权并将控制权交给其他 coroutine。如果开发者无意间或者故意让应用程序长时间占用 CPU，操作系统也无能为力，表现出来的效果就是计算机很容易失去响应或者死机。goroutine 属于**抢占式任务**处理，已经和现有的多线程和多进程任务处理非常类似， 虽然无法控制自己获取高优先度支持。但如果发现一个应用程序长时间大量地占用 CPU，那么用户有权终止这个任务。

### 2、GMP模型

#### 2.1、被淘汰的GM模型

![](http://oss.surfaroundtheworld.top/blog-pictures/6_22/GM.jpg)

M想要执行、放回G都必须访问全局G队列，并且M有多个，即多线程访问同一资源需要加锁进行保证互斥/同步，所以全局G队列是有互斥锁进行保护的。

老调度器有几个缺点：

1. 创建、销毁、调度G都需要每个M获取锁，这就形成了**激烈的锁竞争**。
2. M转移G会造成**延迟和额外的系统负载**。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了**很差的局部性**，因为G’和G是相关的，最好放在M上执行，而不是其他M'。
3. 系统调用(CPU在M之间的切换)导致频繁的线程阻塞和取消阻塞操作增加了系统开销。

#### 2.2、GMP

![](http://oss.surfaroundtheworld.top/blog-pictures/6_22/gpm.png)

其中：

- G：表示 goroutine，每执行一次`go f()`就创建一个 G，包含要执行的函数和上下文信息。
- 全局队列（Global Queue）：存放等待运行的 G。
- P：表示 goroutine 执行所需的资源，最多有 GOMAXPROCS 个。
- P 的本地队列：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建 G 时，G 优先加入到 P 的本地队列，如果本地队列满了会批量移动部分 G 到全局队列。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，当 P 的本地队列为空时，M 也会尝试从全局队列或其他 P 的本地队列获取 G。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。
- Goroutine 调度器和操作系统调度器是通过 M 结合起来的，每个 M 都代表了1个内核线程，操作系统调度器负责把内核线程分配到 CPU 的核上执行。

单从线程调度讲，Go语言相比起其他语言的优势在于OS线程是由OS内核来调度的， goroutine 则是由Go运行时（runtime）自己的调度器调度的，完全是在用户态下完成的， **不涉及内核态与用户态之间的频繁切换**，包括内存的分配与释放，都是在用户态维护着一块大的内存池， 不直接调用系统的malloc函数（除非内存池需要改变），成本比调度OS线程低很多。 另一方面充分利用了多核的硬件资源，近似的把若干goroutine均分在物理线程上， 再加上本身 goroutine 的超轻量级，以上种种特性保证了 goroutine 调度方面的性能。



### 3、垃圾回收机制

#### 1.1、Go的内存管理

程序在内存上被分为**堆区、栈区、全局数据区、代码段、数据区**五个部分。对于C++等早期编程语言栈上的内存由编译器管理回收，堆上的内存空间需要编程人员负责申请与释放。在Go中**栈上内存仍由编译器负责管理回收**，而**堆上的内存由编译器和垃圾收集器负责管理回收**，给编程人员带来了极大的便利性。

#### 1.2、GC的版本史

##### Go1.3的标记清除法

总而言之就是STW时间过长，性能无法满足需求。

##### Go1.5的三色标记法

三色标记算法将程序中的对象分成白色、黑色和灰色三类。白色对象表示暂无对象引用的潜在垃圾，其内存可能会被垃圾收集器回收；灰色对象表示活跃的对象，黑色到白色的中间状态，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；黑色对象表示活跃的对象。

**三色标记法分五步进行**

1. 将所有对象标记为白色
2. 从根节点集合出发，将第一次遍历到的节点标记为灰色放入集合列表中
3. 遍历灰色集合，将灰色节点遍历到的白色节点标记为灰色，并把灰色节点标记为黑色
4. 循环这个过程
5. 直到灰色节点集合为空，回收所有的白色节点

> **根对象是什么？**
>
> 根对象在后面的处理过程中也会进行分类， 不同根对象下的引用堆对象会进行不同的GC标记方式。
>
> 其中栈上的根对象指的是各个协程栈，即G上的栈；
>
> 其他的还有全局变量，全局变量又分为bss段（未初始化的全局变量段）和 data段；
>
> 最后还有span中的finalizer的任务数量，finalizer是Go语言中和某种对象绑定的析构器。当某些对象的内存释放后，需要调用析构器函数，从而完整释放资源。例如os.File对象使用了析构器函数关闭操作系统文件描述符，即便用户忘记了调用close()方法也会释放操作系统资源。

方法本身看起来没有问题，但是当GC和程序并行执行的时候会出现问题。

在此基础上拓展出强三色不变式和弱三色不变式两种方法，对应的分别是插入写屏障和删除写屏障。插入屏障实现的是强三色不变式，删除屏障则实现了弱三色不变式。**值得注意的是为了保证栈的运行效率，屏障只对堆上的内存对象启用，栈上的内存会在GC结束后启用STW重新扫描。**（这里说的栈上的内存，指的是通过栈上指针引用的堆内对象）

插入屏障：对象被引用时触发的机制，当白色对象被黑色对象引用时，白色对象被标记为灰色（栈上对象无插入屏障，因此需要结束只会进行STW重新扫描，STW时间约为10-100ms）。

删除屏障：对象被删除时触发的机制。如果灰色对象引用的白色对象被删除时，那么白色对象会被标记为灰色。这样可以避免删除的对象上还有被引用的对象。

缺点在于回收精度较低，一个对象即使被删除仍可以活过这一轮再下一轮被回收；同样也存在对栈的二次扫描影响程序的效率。

##### Go1.8 三色标记+混合写屏障

基于插入写屏障和删除写屏障在结束时需要STW来重新扫描栈，所带来的性能瓶颈，Go在1.8引入了混合写屏障的方式实现了弱三色不变式的设计方式，混合写屏障，仅在堆空间启动混合写屏障，不需要在`GC`结束后对栈空间重新扫描，`gc pause`时间降低至`0.5ms`以下

流程分下面四步：

1. GC开始时将栈上可达对象全部标记为黑色（不需要二次扫描，无需STW）
2. GC期间，任何栈上创建的新对象均为黑色
3. 被删除引用的对象标记为灰色
4. 被添加引用的对象标记为灰色

##### Go1.14：引入新的页分配器用于优化内存分配的速度

#### 1.3、GC的四个阶段

Golang的GC属于并发式垃圾回收（意味着不需要长时间的STW，GC大部分执行过程是和用户代码并行的），它可以分为四个阶段：

- 清除终止`Sweep Termination`：
  - 暂停程序
  - 清扫未被回收的内存管理单元span，当上一轮GC的清扫工作完成后才可以开始新一轮的GC
- 标记`Mark`：
- - 切换至`_GCmark`，开启写屏障和用户程序协助`Mutator Assiste`并将根对象添加到三色标记法队列
  - 恢复程序，标记进程和`Mutator Assiste`进程会开始并发标记内存中的对象，混合写屏障将被删除的指针和新加入的指针都标记成灰色，新创建的对象标记成黑色
  - 扫描根对象（包括所有goroutine的栈、全局对象以及不在堆中的运行时数据结构），扫描goroutine栈期间会暂停当前处理器
  - 依次处理三色标记法队列，将扫描过的对象标记为黑色并将它们指向的对象标记成灰色
  - 使用分布式终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段
- 标记终止`Mark Termination`
- - 暂停程序，切换至`_GCmarktermination`并关闭辅助标记的用户程序
  - 清理处理器上的线程缓存
- 清除`Sweep`
- - 将状态切换至`_GCoff`，关闭混合写屏障
  - 恢复用户程序，所有新创建的对象标记为白色
  - 后台并发清理所有的内存管理单元span，当goroutine申请新的内存管理单元时就会触发清理

> 在GC过程中会有两种后台任务（G），包括标记任务和清扫任务。可以同时执行的标记任务约是P数量的四分之一，即go所说的25%CPU用于GC的依据。清扫任务会在程序启动后运行，进入清扫阶段时唤醒。当用户分配内存的速度超过GC回收速度时，Golang会通过辅助GC暂停用户程序进行垃圾回收，防止内存因分配对象速度过快消耗殆尽的问题。

#### 1.4、GC触发时机

触发垃圾回收首先要满足三个前提条件：

- `memstats.enablegc`：允许垃圾回收
- `panicking == 0`：程序没有panic
- `gcphase == _GCoff`：处于`_Gcoff`阶段

对应的触发时机包括：

- `gcTriggerHeap`：堆内存的大小达到一定阈值
- `gcTriggerTime`：距离上一次垃圾回收超过一定阈值时
- `gcTriggerCycle`：如果当前没有启动GC则开始新一轮的GC



## 二、语言特性

### 1、闭包

闭包的概念在JS中也有，与go中的性质基本一致。

闭包是由函数及其相关引用环境组合而成的实体(即：闭包=函数+引用环境)。

Go语言是支持闭包的，这里只是简单地讲一下在Go语言中闭包是如何实现的。

```
func f(i int) func() int {
    return func() int {
        i++
        return i
    }
}
```

函数f返回了一个函数，返回的这个函数，返回的这个函数就是一个闭包。这个函数中本身是没有定义变量i的，而是引用了它所在的环境（函数f）中的变量i。

```
c1 := f(0)
c2 := f(0)
c1()    // reference to i, i = 0, return 1
c2()    // reference to another i, i = 0, return 1
```

c1跟c2引用的是不同的环境，在调用i++时修改的不是同一个i，因此两次的输出都是1。函数f每进入一次，就形成了一个新的环境，对应的闭包中，函数都是同一个函数，环境却是引用不同的环境。

变量i是函数f中的局部变量，假设这个变量是在函数f的栈中分配的，是不可以的。因为函数f返回以后，对应的栈就失效了，f返回的那个函数中变量i就引用一个失效的位置了。所以闭包的环境中引用的变量不能够在栈上分配。



在go中有一个比较特别的语言特性：

```
func f() *Cursor {
    var c Cursor
    c.X = 500
    noinline()
    return &c
}
```

以上的写法是不被C语言允许的，因为变量c在栈上分配，f函数返回后c的栈空间失效了，但在go中会对局部变量进行**逃逸分析**，这会将c分配到堆上。

**go在返回闭包时并不是单纯返回一个函数，而是返回了一个结构体，它记录下函数返回地址和引用的环境中的变量地址。**

### 2、defer

Go语言中的`defer`语句会将其后面跟随的语句进行延迟处理。在`defer`归属的函数即将返回时，将延迟处理的语句按`defer`定义的逆序进行执行，也就是说，先被`defer`的语句最后被执行，最后被`defer`的语句，最先被执行。

举个例子：

```go
func main() {
	fmt.Println("start")
	defer fmt.Println(1)
	defer fmt.Println(2)
	defer fmt.Println(3)
	fmt.Println("end")
}
```

输出结果：

```go
start
end
3
2
1
```

由于`defer`语句延迟调用的特性，所以`defer`语句能非常方便的处理资源释放问题。比如：资源清理、文件关闭、解锁及记录时间等。



**defer还有一个特别的注意点：**在Go语言的函数中`return`语句在底层并不是原子操作，它分为给返回值赋值和RET指令两步。而`defer`语句执行的时机就在返回值赋值操作后，RET指令执行前。因此，以下的语句在返回时，返回的实际是5。

```
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```



### 3、Go的并发

Go语言中的并发程序主要是通过基于CSP（communicating sequential processes）的goroutine和channel来实现，当然也支持使用传统的多线程共享内存的并发方式。

#### 3.1、sync.WaitGroup

go中多个goroutine 的并发可以利用sync.WaitGroup很简单地实现：

```go
package main

import (
	"fmt"
	"sync"
)

var wg sync.WaitGroup

func hello(i int) {
	defer wg.Done() // goroutine结束就登记-1
	fmt.Println("hello", i)
}
func main() {
	for i := 0; i < 10; i++ {
		wg.Add(1) // 启动一个goroutine就登记+1
		go hello(i)
	}
	wg.Wait() // 等待所有登记的goroutine都结束
}
```

#### 3.2、channel

上面的方式只能将函数进行并发，但是没有数据通信，go的并发模型是`CSP（Communicating Sequential Processes）`，提倡**通过通信共享内存**而不是**通过共享内存而实现通信**。go中使用channel作为goroutine之间的通信机制。

声明的通道类型变量需要使用内置的`make`函数初始化之后才能使用。具体格式如下：

```go
make(chan 元素类型, [缓冲大小])
```

其中：

- channel的缓冲大小是可选的。

##### 无缓冲通道

作用和WaitGroup基本一致，示例如下：

```go
func recv(c chan int) {
	ret := <-c
	fmt.Println("接收成功", ret)
}

func main() {
	ch := make(chan int)
	go recv(ch) // 创建一个 goroutine 从通道接收值
	ch <- 10
	fmt.Println("发送成功")
}
```

首先无缓冲通道`ch`上的发送操作会阻塞，直到另一个 goroutine 在该通道上执行接收操作，这时数字10才能发送成功，两个 goroutine 将继续执行。相反，如果接收操作先执行，接收方所在的 goroutine 将阻塞，直到 main goroutine 中向该通道发送数字10。

使用无缓冲通道进行通信将导致发送和接收的 goroutine 同步化。因此，无缓冲通道也被称为**同步通道**。

##### 有缓冲的通道

只要通道的容量大于零，那么该通道就属于有缓冲的通道，通道的容量表示通道中最大能存放的元素数量。当通道内已有元素数达到最大容量后，再向通道执行发送操作就会阻塞，除非有从通道执行接收操作。

##### 如何判断一个通道是否被关闭？

对一个通道执行接收操作时支持使用如下多返回值模式。

```go
value, ok := <- ch
```

其中：

- value：从通道中取出的值，如果通道被关闭则返回对应类型的零值。
- ok：通道ch关闭时返回 false，否则返回 true。

##### for range接收值

通常我们会选择使用`for range`循环从通道中接收值，当通道被关闭后，会在通道内的所有值被接收完毕后会自动退出循环。这样可以无需判断ok是否为false。

```go
func f3(ch chan int) {
	for v := range ch {
		fmt.Println(v)
	}
}
```

##### 单向通道

```go
<- chan int // 只接收通道，只能接收不能发送
chan <- int // 只发送通道，只能发送不能接收
```

其中，箭头`<-`和关键字`chan`的相对位置表明了当前通道允许的操作，这种限制将在编译阶段进行检测。另外对一个只接收通道执行close也是不允许的，因为**默认通道的关闭操作应该由发送方来完成**。

```go
// Producer2 返回一个接收通道
func Producer2() <-chan int {
	ch := make(chan int, 2)
	// 创建一个新的goroutine执行发送数据的任务
	go func() {
		for i := 0; i < 10; i++ {
			if i%2 == 1 {
				ch <- i
			}
		}
		close(ch) // 任务完成后关闭通道
	}()

	return ch
}

// Consumer2 参数为接收通道
func Consumer2(ch <-chan int) int {
	sum := 0
	for v := range ch {
		sum += v
	}
	return sum
}

func main() {
	ch2 := Producer2()
  
	res2 := Consumer2(ch2)
	fmt.Println(res2) // 25
}
```

#### 3.3、锁机制

提供了互斥锁，读写锁等，操作和其他编程语言基本一样。

#### 3.4、sync.Once

在某些场景下我们需要确保某些操作即使在高并发的场景下也只会被执行一次，例如只加载一次配置文件等。

Go语言中的`sync`包中提供了一个针对只执行一次场景的解决方案——`sync.Once`.

这是实现并发安全的单例模式的一种方法：

```go
package singleton

import (
    "sync"
)

type singleton struct {}

var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
```

`sync.Once`其实内部包含一个互斥锁和一个布尔值，互斥锁保证布尔值和数据的安全，而布尔值用来记录初始化是否完成。这样设计就能保证初始化操作的时候是并发安全的并且初始化操作也不会被执行多次。

#### 3.5、原子操作

针对整数数据类型（int32、uint32、int64、uint64）我们还可以使用原子操作来保证并发安全，通常直接使用原子操作比使用锁操作效率更高。Go语言中原子操作由内置的标准库`sync/atomic`提供。

`atomic`包提供了底层的原子级内存操作，对于同步算法的实现很有用。这些函数必须谨慎地保证正确使用。**除了某些特殊的底层应用，使用通道或者 sync 包的函数/类型实现同步更好。**

### 4、Context

因为在写项目中，先使用了gin中的context，因此看到Go标准库中对Context的介绍让我迷惑了好久。后来发现，好像这是两个功能不同的东西？

这里整理一下标准库中的Context的作用。它是专门用来简化 对于处理单个请求的多个 goroutine 之间与请求域的数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 API 调用。

对服务器传入的请求应该创建上下文，而对服务器的传出调用应该接受上下文。它们之间的函数调用链必须传递上下文，或者可以使用`WithCancel`、`WithDeadline`、`WithTimeout`或`WithValue`创建的派生上下文。当一个上下文被取消时，它派生的所有上下文也被取消。

`context`包中还定义了四个With系列函数。分别是WithCancel、WithDeadline、WithTimeout和WithValue。其使用方法也都大同小异。

以WithCancel为例：

取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用cancel。

```go
func gen(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return // return结束该goroutine，防止泄露
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // 当我们取完需要的整数后调用cancel

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}
```

上面的示例代码中，`gen`函数在单独的goroutine中生成整数并将它们发送到返回的通道。 gen的调用者在使用生成的整数之后需要取消上下文，以免`gen`启动的内部goroutine发生泄漏。

### 5、go中的"继承"与多态

Go语言中没有类的概念，也不支持类的继承等面向对象的概念。Go中通过结构体内嵌和接口两种方式实现了具有更高扩展性和灵活性的功能。

#### 5.1、go只有组合没有继承

go中利用了结构体的组合实现了其他语言中继承的功能，本质上是一种语法糖：

```
//Animal 动物
type Animal struct {
	name string
}

func (a *Animal) move() {
	fmt.Printf("%s会动！\n", a.name)
}

//Dog 狗
type Dog struct {
	Feet    int8
	*Animal //通过嵌套匿名结构体实现继承
}

func (d *Dog) wang() {
	fmt.Printf("%s会汪汪汪~\n", d.name)
}

func main() {
	d1 := &Dog{
		Feet: 4,
		Animal: &Animal{ //注意嵌套的是结构体指针
			name: "乐乐",
		},
	}
	d1.wang() //乐乐会汪汪汪~
	d1.move() //乐乐会动！
}
```

### 5.2、通过组合实现多态

多态指的是代码可以根据类型的不同执行不同的操作。

一个类型可以实现多个接口，多个接口实现实现同一类型，根据实际可以灵活应用。

```
/* 多态行为的例子 */
package main

import "fmt"

type notifer interface {
   notify()
}

type user struct {
   name string
   email string
}

type admin struct {
   name string
   age  int
}

type ad struct {
   name string
   age  int
}

//使用指针接收者实现了notofy接口,方法会共享接收者所指向的值user
func (u *user) notify()  {
   fmt.Println("sendNotify to user", u.name)
}

//使用值接收者实现了notify接口,方法使用u值的副本,对u的修改不会影响原值
func (u admin) notify(){
   fmt.Println("sendNotify to admin:", u.name)
}

//接收一个notifer接口类型的值，如果一个实体类型实现了该接口，
//sendNotify函数会根据实体类型的值类执行notifer接口的notify行为，这个函数具有多态的能力。
func sendNotify(n notifer){
   n.notify()
}

func main()  {
   user1 := user{"张三", "qq@qq.com"}
   sendNotify(&user1)

   user2 := admin{"李四", 25}
   sendNotify(user2)

   var n notifer
   n = &user1
   n.notify()
}
```



## 三、使用小技巧

### 1、空 struct{} 的用途

使用空结构体 struct{} 可以节省内存，一般作为占位符使用，表明这里并不需要一个值。

比如使用 map 表示集合时，只关注 key，value 可以使用 struct{} 作为占位符。如果使用其他类型作为占位符，例如 int，bool，不仅浪费了内存，而且容易引起歧义。

```
type Set map[string]struct{}

func main() {
	set := make(Set)

	for _, item := range []string{"A", "A", "B", "C"} {
		set[item] = struct{}{}
	}
	fmt.Println(len(set)) // 3
	if _, ok := set["A"]; ok {
		fmt.Println("A exists") // A exists
	}
}
```

再比如，使用信道(channel)控制并发时，我们只是需要一个信号，但并不需要传递值，这个时候，也可以使用 struct{} 代替。

```
func main() {
	ch := make(chan struct{}, 1)
	go func() {
		<-ch
		// do something
	}()
	ch <- struct{}{}
	// ...
}
```

再比如，声明只包含方法的结构体。

```
type Lamp struct{}

func (l Lamp) On() {
        println("On")

}
func (l Lamp) Off() {
        println("Off")
}
```

### 2、make和new

二者都是内存的分配（堆上），但是`make`只用于slice、map以及channel的初始化（非零值）；而`new`用于类型的内存分配，并且内存置为零。

`make`返回的还是这三个引用类型本身；而`new`返回的是指向类型的指针。

但是new不常用，可以直接用`:=`替代，而make不能被替代。

make的slice用法：

```bash
make([]T, size, cap)
```

其中：

- T:切片的元素类型
- size:切片中元素的数量
- cap:切片的容量

### 3、Timer

### 4、sync.Cond的使用

# 参考文章

1、[图解Golang垃圾回收机制！](https://zhuanlan.zhihu.com/p/390926887)

2、[图示Golang垃圾回收机制](https://zhuanlan.zhihu.com/p/297177002)

3、[Python 协程和 Go 协程有什么区别？](https://zhuanlan.zhihu.com/p/408759905)

4、[[典藏版]Golang调度器GPM原理与调度全分析](https://zhuanlan.zhihu.com/p/323271088)

5、https://www.liwenzhou.com/posts/Go/concurrence/

6、[闭包的实现](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.6.html)

7、[【Golang】脱胎换骨的defer](https://mp.weixin.qq.com/s/gaC2gmFhJezH-9-uxpz07w)