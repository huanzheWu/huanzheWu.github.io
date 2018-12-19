---
layout:     post
title:      读书：《Go in action》
subtitle:   聊聊golang的并发与并行
date:       2018-09-23
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 读书
    - Golang
---


[TOC]

本文的主要内容是：

> * 了解goroutine，使用它来运行程序
> * 了解Go是如何检测并修正竞争状态的（解决资源互斥访问的方式）
> * 了解并使用通道chan来同步goroutine


# 一、使用goroutine来运行程序
### 1.Go的并发与并行

Go的并发能力，是指让某个函数独立于其他函数运行的能力。当为一个函数创建`goroutine`时，该函数将作为一个独立的工作单元，被 调度器 调度到可用的逻辑处理器上执行。Go的运行时调度器是个复杂的软件，它做的工作大致是：

- 管理被创建的所有goroutine，为其分配执行时间
- 将操作系统线程与语言运行时的逻辑处理器绑定

参考[The Go scheduler](http://morsmachine.dk/go-scheduler)  ，这里较浅显地说一下Go的运行时调度器。操作系统会在物理处理器上调度`操作系统线程`来运行，而Go语言的运行时会在`逻辑处理器`上调度`goroutine`来运行，每个逻辑处理器都分别绑定到单个操作系统线程上。这里涉及到三个角色：
- M：操作系统线程，这是真正的内核OS线程
- P：逻辑处理器，代表着调度的上下文，它使goroutine在一个M上跑
- G：goroutine，拥有自己的栈，指令指针等信息，被P调度

![](http://pj05m6t8l.bkt.clouddn.com/8_1.jpg)

每个P会维护一个全局运行队列（称为runqueue），处于ready就绪状态的`goroutine`（灰色G）被放在这个队列中等待被调度。在编写程序时，每当`go func`启动一个`goroutine`时，`runqueue`便在尾部加入一个`goroutine`。在下一个调度点上，P就从`runqueue`中取出一个`goroutine`出来执行（蓝色G）。

当某个操作系统线程M阻塞的时候（比如`goroutine`执行了阻塞的系统调用)，P可以绑定到另外一个操作系统线程M上，让运行队列中的其他`goroutine`继续执行：

 ![](http://pj05m6t8l.bkt.clouddn.com/8_2.jpg)

上图中G0执行了阻塞操作，M0被阻塞，P将在新的系统线程M1上继续调度G执行。M1有可能是被新创建的，或者是从线程缓存中取出。Go调度器保证有足够的线程来运行所有的P，语言运行时默认限制每个程序最多创建10000个线程，这个现在可以通过调用runtime/debug包的`SetMaxThreads`方法来更改。

Go可以在在一个逻辑处理器P上实现并发，如果需要并行，必须使用多于1个的逻辑处理器。Go调度器会把`goroutine`平等分配到每个逻辑处理器上，此时`goroutine`将在不同的线程上运行，不过前提是要求机器拥有多个物理处理器。

### 2.创建goroutine
使用关键字`go`来创建一个`goroutine`，并让所有的`goroutine`都得到执行：
```
//example1.go
package main

import (
   "runtime"
   "sync"
   "fmt"
)

var (
   wg sync.WaitGroup
)

func main() {
   //分配一个逻辑处理器Ｐ给调度器使用
   runtime.GOMAXPROCS(1)
   //在这里,wg用于等待程序完成，计数器加2，表示要等待两个goroutine
   wg.Add(2)
   //声明1个匿名函数，并创建一个goroutine
   fmt.Printf("Begin Coroutines\n")
   go func() {
      //在函数退出时，wg计数器减1
      defer wg.Done()
      //打印3次小写字母表
      for count := 0; count < 3; count++ {
         for char := 'a'; char < 'a'+26; char++ {
            fmt.Printf("%c ", char)
         }
      }
   }()
   //声明1个匿名函数，并创建一个goroutine
   go func() {
      defer wg.Done()
      //打印大写字母表3次
      for count := 0; count < 3; count++ {
         for char := 'A'; char < 'A'+26; char++ {
            fmt.Printf("%c ", char)
         }
      }
   }()

   fmt.Printf("Waiting To Finish\n")
   //等待2个goroutine执行完毕
   wg.Wait()

}

```
这个程序使用` runtime.GOMAXPROCS(1)`来分配一个逻辑处理器给调度器使用，两个`goroutine`将被该逻辑处理器调度并发执行。程序输出：
```
Begin Coroutines
Waiting To Finish
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z 

```
从输出来看，是先执行完一个`goroutine`，再接着执行第二个`goroutine`的，大写字母全部打印完后，再打印全部的小写字母。那么，**有没有办法让两个`goroutine`并行执行呢？**为程序指定两个逻辑处理器即可：
```
//修改为2个逻辑处理器
runtime.GOMAXPROCS(2)
```
此时执行程序，输出为：
```
Begin Coroutines
Waiting To Finish
A B C D E a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z a b c F G H I J K L M N O P Q R S T U V W X d e f g h i j k l m n o p q r s Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z t u v w x y z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z 
```
**那如果只有1个逻辑处理器，如何让两个goroutine交替被调度？**实际上，如果`goroutine`需要很长的时间才能运行完，调度器的内部算法会将当前运行的`goroutine`让出，防止某个`goroutine`长时间占用逻辑处理器。由于示例程序中两个`goroutine`的执行时间都很短，在为引起调度器调度之前已经执行完。不过，程序也可以使用`runtime.Gosched()`来将当前在逻辑处理器上运行的`goruntine`让出，让另一个`goruntine`得到执行：
```
//example2.go
package main

import (
   "runtime"
   "sync"
   "fmt"
)

var (
   wg sync.WaitGroup
)

func main() {
   //分配一个逻辑处理器Ｐ给调度器使用
   runtime.GOMAXPROCS(1)
   //在这里,wg用于等待程序完成，计数器加2，表示要等待两个goroutine
   wg.Add(2)
   //声明1个匿名函数，并创建一个goroutine
   fmt.Printf("Begin Coroutines\n")
   go func() {
      //在函数退出时，wg计数器减1
      defer wg.Done()
      //打印3次小写字母表
      for count := 0; count < 3; count++ {
         for char := 'a'; char < 'a'+26; char++ {
            if char=='k'{
               runtime.Gosched()
            }
            fmt.Printf("%c ", char)
         }
      }
   }()
   //声明1个匿名函数，并创建一个goroutine
   go func() {
      defer wg.Done()
      //打印大写字母表3次
      for count := 0; count < 3; count++ {
         for char := 'A'; char < 'A'+26; char++ {
            if char == 'K'{
               runtime.Gosched()
            }
            fmt.Printf("%c ", char)
         }
      }
   }()

   fmt.Printf("Waiting To Finish\n")
   //等待2个goroutine执行完毕
   wg.Wait()

}

```
两个`goroutine`在循环的字符为k/K的时候会让出逻辑处理器，程序的输出结果为：
```
Begin Coroutines
Waiting To Finish
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J a b c d e f g h i j K L M N O P Q R S T U V W X Y Z A B C D E F G H I J k l m n o p q r s t u v w x y z a b c d e f g h i j K L M N O P Q R S T U V W X Y Z k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z 
```
这里大小写字母果然是交替着输出了。不过从输出可以看到，第一次输出大写字母时遇到K没有让出逻辑处理器，这是什么原因还不是很清楚，调度器的调度机制？


# 二、处理竞争状态
并发程序避免不了的一个问题是对资源的同步访问。如果多个`goroutine`在没有互相同步的情况下去访问同一个资源，并进行读写操作，这时`goroutine`就处于竞争状态下：
```
//example3.go
package main

import (
   "sync"
   "runtime"
   "fmt"
)

var (
   //counter为访问的资源
   counter int64
   wg      sync.WaitGroup
)

func addCount() {
   defer wg.Done()
   for count := 0; count < 2; count++ {
      value := counter
      //当前goroutine从线程退出
      runtime.Gosched()
      value++
      counter=value
   }
}

func main() {
   wg.Add(2)
   go addCount()
   go addCount()
   wg.Wait()
   fmt.Printf("counter: %d\n",counter)
}
//output:
counter: 4 或者counter: 2
```
这段程序中，`goroutine`对`counter`的读写操作没有进行同步，goroutine 1对counter的写结果可能被goroutine 2所覆盖。Go可通过如下方式来解决这个问题：
- 使用原子函数操作
- 使用互斥锁锁住临界区
- 使用通道`chan`


### 1. 检测竞争状态
有时候竞争状态并不能一眼就看出来。Go 提供了一个非常有用的工具，用于检测竞争状态。使用方式是：
> go build -race example4.go//用竞争检测器标志来编译程序
> ./example4  //运行程序

![](http://pj05m6t8l.bkt.clouddn.com/8_3.png)

 工具检测出了程序存在一处竞争状态，并指出发生竞争状态的几行代码是：
```
22        counter=value
18        value := counter
28        go addCount()
29        go addCount()
```

### 2. 使用原子函数
对整形变量或指针的同步访问，可以使用原子函数来进行。这里使用原子函数来修复example4.go中的竞争状态问题：
```
//example5.go
package main

import (
    "sync"
    "runtime"
    "fmt"
    "sync/atomic"
)

var (
    //counter为访问的资源
    counter int64
    wg      sync.WaitGroup
)

func addCount() {
    defer wg.Done()
    for count := 0; count < 2; count++ {
        //使用原子操作来进行
        atomic.AddInt64(&counter,1)
        //当前goroutine从线程退出
        runtime.Gosched()
    }
}

func main() {
    wg.Add(2)
    go addCount()
    go addCount()
    wg.Wait()
    fmt.Printf("counter: %d\n",counter)
}
//output：
counter: 4
```
这里使用`atomic.AddInt64`函数来对一个整形数据进行加操作，另外一些有用的原子操作还有：
```
atomic.StoreInt64() //写
atomic.LoadInt64()  //读
```
更多的原子操作函数请看`atomic`包中的声明。

### 3. 使用互斥锁
对临界区的访问，可以使用互斥锁来进行。对于example4.go的竞争状态，可以使用互斥锁来解决：
```
//example5.go
package main

import (
   "sync"
   "runtime"
   "fmt"
)

var (
   //counter为访问的资源
   counter int
   wg      sync.WaitGroup
   mutex   sync.Mutex
)

func addCount() {
   defer wg.Done()

   for count := 0; count < 2; count++ {
      //加上锁，进入临界区域
      mutex.Lock()
      {
         value := counter
         //当前goroutine从线程退出
         runtime.Gosched()
         value++
         counter = value
      }
      //离开临界区，释放互斥锁
      mutex.Unlock()
   }
}
func main() {
   wg.Add(2)
   go addCount()
   go addCount()
   wg.Wait()
   fmt.Printf("counter: %d\n", counter)
}

//output:
counter: 4
```
使用`Lock()`与`Unlock()`函数调用来定义临界区，在同一个时刻内，只有一个goroutine能够进入临界区，直到调用`Unlock()`函数后，其他的goroutine才能够进入临界区。

在Go中解决共享资源安全访问，更常用的使用通道chan。

# 三、利用通道共享数据
Go语言采用CSP消息传递模型。通过在`goroutine`之间传递数据来传递消息，而不是对数据进行加锁来实现同步访问。这里就需要用到通道`chan`这种特殊的数据类型。当一个资源需要在`goroutine`中共享时，chan在`goroutine`中间架起了一个通道。通道使用`make`来创建：
```
unbuffered := make(char int) //创建无缓存通道，用于int类型数据共享
buffered := make(chan string,10)//创建有缓存通道，用于string类型数据共享
buffered<- "hello world" //向通道中写入数据
value:= <-buffered //从通道buffered中接受数据
```
通道用于放置某一种类型的数据。创建通道时指定通道的大小，将创建有缓存的通道。无缓存通道是一种同步通信机制，它要求发送`goroutine`和接收`goroutine`都应该准备好，否则会进入阻塞。

###1. 无缓存的通道
无缓存通道是同步的——一个`goroutine`向channel写入消息的操作会一直阻塞，直到另一个`goroutine`从通道中读取消息。反过来也是，一个`goroutine`从channel读取消息的操作会一直阻塞，直到另一个`goroutine`向通道中写入消息。《Go in action》中关于无缓存通道的解释有一个非常棒的例子：网球比赛。在网球比赛中，两位选手总是处在以下两种状态之一：要么在等待接球，要么在把球打向对方。球的传递可看为通道中数据传递。下面这段代码使用通道模拟了这个过程：
```
//example6.go
package main

import (
    "sync"
    "fmt"
    "math/rand"
    "time"
)

var wg sync.WaitGroup

func player(name string, court chan int) {
    defer wg.Done()
    for {
        //如果通道关闭,那么选手胜利
        ball, ok := <-court
        if !ok {
            fmt.Printf("Player %s Won\n", name)
            return
        }
        n := rand.Intn(100)

        //随机概率使某个选手Miss
        if n%13 == 0 {
            fmt.Printf("Player %s Missed\n", name)
            //关闭通道
            close(court)
            return
        }
        fmt.Printf("Player %s Hit %d\n", name, ball)
        ball++
        //否则选手进行击球
        court <- ball
    }
}


func main() {
    rand.Seed(time.Now().Unix())
    court := make(chan int)
    //等待两个goroutine都执行完
    wg.Add(2)
    //选手1等待接球
    go player("candy", court)
    //选手2等待接球
    go player("luffic", court)
    //球进入球场（可以开始比赛了）
    court <- 1
    wg.Wait()
}
//output:
Player luffic Hit 1
Player candy Hit 2
Player luffic Hit 3
Player candy Hit 4
Player luffic Hit 5
Player candy Missed
Player luffic Won
```



### 2. 有缓存的通道
有缓存的通道是一种在被接收前能存储一个或者多个值的通道，它与无缓存通道的区别在于：无缓存的通道保证进行发送和接收的goroutine会在同一时间进行数据交换，有缓存的通道没有这种保证。有缓存通道让`goroutine`阻塞的条件为：通道中没有数据可读的时候，接收动作会被阻塞；通道中没有区域容纳更多数据时，发送动作阻塞。向已经关闭的通道中发送数据，会引发panic，但是`goroutine`依旧能从通道中接收数据，但是不能再向通道里发送数据。所以，发送端应该负责把通道关闭，而不是由接收端来关闭通道。

# 小结
- goroutine被逻辑处理器执行，逻辑处理器拥有独立的系统线程与运行队列
- 多个goroutine在一个逻辑处理器上可以并发执行，当机器有多个物理核心时，可通过多个逻辑处理器来并行执行。
- 使用关键字 go 来创建goroutine。
- 在Go中，竞争状态出现在多个goroutine试图同时去访问一个资源时。
- 可以使用互斥锁或者原子函数，去防止竞争状态的出现。
- 在go中，更好的解决竞争状态的方法是使用通道来共享数据。
- 无缓冲通道是同步的，而有缓冲通道不是。

（完）






