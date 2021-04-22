---
title: "学习笔记-Go语言高级编程-语言基础"
date: 2021-04-19T15:16:43+08:00
draft: false
auther: "大脸波"
tags: ["go","golang","学习笔记"]
---

### 数组

   数组是一个由固定长度的特定类型元素组成的序列，一个数组可以由零个或多个元素组成。和数组对应的类型是切片，切片是可以动态增长和收缩的序列，切片的功能也更加灵活，但是要理解切片的工作原理还是要先理解数组。

   数组的定义方式:
   ```go
   var a [3]int                    // 定义长度为3的int型数组, 元素全部为0
   var b = [...]int{1, 2, 3}       // 定义长度为3的int型数组, 元素为 1, 2, 3
   var c = [...]int{2: 3, 1: 2}    // 定义长度为3的int型数组, 元素为 0, 2, 3
   var d = [...]int{1, 2, 4: 5, 6} // 定义长度为6的int型数组, 元素为 1, 2, 0, 0, 5, 6
   ```

   当一个数组变量被赋值或者被传递的时候，实际上会复制整个数组。如果数组较大的话，数组的赋值也会有较大的开销。为了避免复制数组带来的开销，可以传递一个指向数组的指针，但是数组指针并不是数组。

   我们可以用for循环来迭代数组。下面常见的几种方式都可以用来遍历数组：
   ```go
   for i := range a {
        fmt.Printf("a[%d]: %d\n", i, a[i])
    }
    for i, v := range b {
        fmt.Printf("b[%d]: %d\n", i, v)
    }
    for i := 0; i < len(c); i++ {
        fmt.Printf("c[%d]: %d\n", i, c[i])
    }
    ```

   我们还可以定义一个空的数组：
   ```go
   var d [0]int       // 定义一个长度为0的数组
   var e = [0]int{}   // 定义一个长度为0的数组
   var f = [...]int{} // 定义一个长度为0的数组
   ```
   长度为0的数组在内存中并不占用空间。空数组虽然很少直接使用，但是可以用于强调某种特有类型的操作时避免分配额外的内存空间，比如用于管道的同步操作：
   ```go
   c1 := make(chan [0]int)
    go func() {
        fmt.Println("c1")
        c1 <- [0]int{}
    }()
    <-c1
    ```
    在这里，我们并不关心管道中传输数据的真实类型，其中管道接收和发送操作只是用于消息的同步。对于这种场景，我们用空数组来作为管道类型可以减少管道元素赋值时的开销。当然一般更倾向于用无类型的匿名结构体代替：
    ```go
    c2 := make(chan struct{})
   go func() {
       fmt.Println("c2")
       c2 <- struct{}{} // struct{}部分是类型, {}表示对应的结构体值
   }()
   <-c2
   ```

### 字符串

   一个字符串是一个不可改变的字节序列，字符串通常是用来包含人类可读的文本数据。和数组不同的是，字符串的元素不可修改，是一个只读的字节数组。每个字符串的长度虽然也是固定的，但是字符串的长度并不是字符串类型的一部分。由于Go语言的源代码要求是UTF8编码，导致Go源代码中出现的字符串面值常量一般也是UTF8编码的。源代码中的文本字符串通常被解释为采用UTF8编码的Unicode码点（rune）序列。

   Go语言字符串的底层结构在reflect.StringHeader中定义：
   ```go
   type StringHeader struct {
      Data uintptr
      Len  int
   }
   ```
   字符串结构由两个信息组成：第一个是字符串指向的底层字节数组，第二个是字符串的字节的长度。

   字符串虽然不是切片，但是支持切片操作，不同位置的切片底层也访问的同一块内存数据（因为字符串是只读的，相同的字符串面值常量通常是对应同一个字符串常量）：
   ```go
   s := "hello, world"
   hello := s[:5]
   world := s[7:]

   s1 := "hello, world"[:5]
   s2 := "hello, world"[7:]
   ```
   字符串相关的强制类型转换主要涉及到[]byte和[]rune两种类型。每个转换都可能隐含重新分配内存的代价，最坏的情况下它们的运算时间复杂度都是O(n)。不过字符串和[]rune的转换要更为特殊一些，因为一般这种强制类型转换要求两个类型的底层内存结构要尽量一致，显然它们底层对应的[]byte和[]int32类型是完全不同的内部布局，因此这种转换可能隐含重新分配内存的操作。

### 切片(slice)

   简单地说，切片就是一种简化版的动态数组。因为动态数组的长度是不固定，切片的长度自然也就不能是类型的组成部分了。数组虽然有适用它们的地方，但是数组的类型和操作都不够灵活，因此在Go代码中数组使用的并不多。而切片则使用得相当广泛，理解切片的原理和用法是一个Go程序员的必备技能。

   我们先看看切片的结构定义，reflect.SliceHeader：
   ```go
   type SliceHeader struct {
      Data uintptr
      Len  int
      Cap  int
   }
   ```
   可以看出切片的开头部分和Go字符串是一样的，但是切片多了一个Cap成员表示切片指向的内存空间的最大容量（对应元素的个数，而不是字节数）。

   让我们看看切片有哪些定义方式：
   ```go
   var (
      a []int               // nil切片, 和 nil 相等, 一般用来表示一个不存在的切片
      b = []int{}           // 空切片, 和 nil 不相等, 一般用来表示一个空的集合
      c = []int{1, 2, 3}    // 有3个元素的切片, len和cap都为3
      d = c[:2]             // 有2个元素的切片, len为2, cap为3
      e = c[0:2:cap(c)]     // 有2个元素的切片, len为2, cap为3
      f = c[:0]             // 有0个元素的切片, len为0, cap为3
      g = make([]int, 3)    // 有3个元素的切片, len和cap都为3
      h = make([]int, 2, 3) // 有2个元素的切片, len为2, cap为3
      i = make([]int, 0, 3) // 有0个元素的切片, len为0, cap为3
   )
   ```
   和数组一样，内置的len函数返回切片中有效元素的长度，内置的cap函数返回切片容量大小，容量必须大于或等于切片的长度。也可以通过reflect.SliceHeader结构访问切片的信息（只是为了说明切片的结构，并不是推荐的做法）。

   遍历切片的方式和遍历数组的方式类似：
   ```go
   for i := range a {
        fmt.Printf("a[%d]: %d\n", i, a[i])
    }
    for i, v := range b {
        fmt.Printf("b[%d]: %d\n", i, v)
    }
    for i := 0; i < len(c); i++ {
        fmt.Printf("c[%d]: %d\n", i, c[i])
    }
   ```
   内置的泛型函数append可以在切片的尾部追加N个元素：
   ```go
   var a []int
   a = append(a, 1)               // 追加1个元素
   a = append(a, 1, 2, 3)         // 追加多个元素, 手写解包方式
   a = append(a, []int{1,2,3}...) // 追加一个切片, 切片需要解包
   ```
   不过要注意的是，在容量不足的情况下，append的操作会导致重新分配内存，可能导致巨大的内存分配和复制数据代价。即使容量足够，依然需要用append函数的返回值来更新切片本身，因为新切片的长度已经发生了变化。

   删除切片元素：
   ```go
   a = []int{1, 2, 3}
   a = a[:len(a)-1]   // 删除尾部1个元素
   a = a[:len(a)-N]   // 删除尾部N个元素
   a = a[1:]          // 删除开头1个元素
   a = a[N:]          // 删除开头N个元素

   a = a[:copy(a, a[1:])] // 删除开头1个元素
   a = a[:copy(a, a[N:])] // 删除开头N个元素

   a = append(a[:i], a[i+1:]...) // 删除中间1个元素
   a = append(a[:i], a[i+N:]...) // 删除中间N个元素

   a = a[:i+copy(a[i:], a[i+1:])]  // 删除中间1个元素
   a = a[:i+copy(a[i:], a[i+N:])]  // 删除中间N个元素
   ```

   假设切片里存放的是指针对象，那么下面删除末尾的元素后，被删除的元素依然被切片底层数组引用，从而导致不能及时被自动垃圾回收器回收（这要依赖回收器的实现方式）：
   ```go
   var a []*int{ ... }
   a = a[:len(a)-1]    // 被删除的最后一个元素依然被引用, 可能导致GC操作被阻碍

   //可以改成
   a[len(a)-1] = nil // GC回收最后一个元素内存
   a = a[:len(a)-1]  // 从切片删除最后一个元素
   ```
### 函数、方法和接口

   在Go语言中，函数是第一类对象，我们可以将函数保持到变量中。函数主要有具名和匿名之分，包级函数一般都是具名函数，具名函数是匿名函数的一种特例。当然，Go语言中每个类型还可以有自己的方法，方法其实也是函数的一种。
   ```go
   // 具名函数
   func Add(a, b int) int {
       return a+b
   }

   // 匿名函数
   var Add = func(a, b int) int {
       return a+b
   }

   // 多个参数和多个返回值
   func Swap(a, b int) (int, int) {
       return b, a
   }

   // 可变数量的参数
   // more 对应 []int 切片类型
   func Sum(a int, more ...int) int {
       for _, v := range more {
           a += v
       }
       return a
   }

   // 不仅函数的参数可以有名字，也可以给函数的返回值命名
   func Find(m map[int]int, key int) (value int, ok bool) {
      value, ok = m[key]
      return
   }
   ```
   闭包的这种引用方式访问外部变量的行为可能会导致一些隐含的问题：
   ```go
   func main() {
      for i := 0; i < 3; i++ {
          defer func(){ println(i) } ()
      }
   }
   // Output:
   // 3
   // 3
   // 3

   //可以通过下面2中方式修复
   func main() {
       for i := 0; i < 3; i++ {
           i := i // 定义一个循环体内局部变量i
           defer func(){ println(i) } ()
       }
   }

   func main() {
       for i := 0; i < 3; i++ {
           // 通过函数传入i
           // defer 语句会马上对调用参数求值
           defer func(i int){ println(i) } (i)
       }
   }
   ```

   方法一般是面向对象编程(OOP)的一个特性，面向对象编程更多的只是一种思想，很多号称支持面向对象编程的语言只是将经常用到的特性内置到语言中了而已。
   ```go
   // 关闭文件
   func (f *File) Close() error {
       // ...
   }

   // 读文件数据
   func (f *File) Read(offset int64, data []byte) int {
       // ...
   }
   ```
   方法是由函数演变而来，只是将函数的第一个对象参数移动到了函数名前面了而已。从代码角度看虽然只是一个小的改动，但是从编程哲学角度来看，Go语言已经是进入面向对象语言的行列了。我们可以给任何自定义类型添加一个或多个方法。每种类型对应的方法必须和类型的定义在同一个包中。对于给定的类型，每个方法的名字必须是唯一的，同时方法和函数一样也不支持重载。

   Go语言不支持传统面向对象中的继承特性，而是以自己特有的组合方式支持了方法的继承。Go语言中，通过在结构体内置匿名的成员来实现继承。
   ```go
   import "image/color"

   type Point struct{ X, Y float64 }

   type ColoredPoint struct {
       Point
       Color color.RGBA
   }

   var cp ColoredPoint
   cp.X = 1
   fmt.Println(cp.Point.X) // "1"
   cp.Point.Y = 2
   fmt.Println(cp.Y)       // "2"
   ```

   通过嵌入匿名的成员，我们不仅可以继承匿名成员的内部成员，而且可以继承匿名成员类型所对应的方法。
   ```go
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

   Go的接口类型是对其它类型行为的抽象和概括；因为接口类型不会和特定的实现细节绑定在一起，通过这种抽象的方式我们可以让对象更加灵活和更具有适应能力。很多面向对象的语言都有相似的接口概念，但Go语言中接口类型的独特之处在于它是满足隐式实现的鸭子类型。所谓鸭子类型说的是：只要走起路来像鸭子、叫起来也像鸭子，那么就可以把它当作鸭子。Go语言中的面向对象就是如此，如果一个对象只要看起来像是某种接口类型的实现，那么它就可以作为该接口类型使用。这种设计可以让你创建一个新的接口类型满足已经存在的具体类型却不用去破坏这些类型原有的定义；当我们使用的类型来自于不受我们控制的包时这种设计尤其灵活有用。Go语言的接口类型是延迟绑定，可以实现类似虚函数的多态功能。

   ```go
   package main

   import (
       "fmt"
       "testing"
   )

   type TB struct {
       testing.TB
   }

   func (p *TB) Fatal(args ...interface{}) {
       fmt.Println("TB.Fatal disabled!")
   }

   func main() {
       var tb testing.TB = new(TB)
       tb.Fatal("Hello, playground")
   }
   ```
   我们在自己的TB结构体类型中重新实现了Fatal方法，然后通过将对象隐式转换为testing.TB接口类型（因为内嵌了匿名的testing.TB对象，因此是满足testing.TB接口的），然后通过testing.TB接口来调用我们自己的Fatal方法。

### 面向并发的内存模型

   Goroutine是Go语言特有的并发体，是一种轻量级的线程，由go关键字启动。在真实的Go语言的实现中，goroutine和系统线程也不是等价的。尽管两者的区别实际上只是一个量的区别，但正是这个量变引发了Go语言并发编程质的飞跃。

   首先，每个系统级线程都会有一个固定大小的栈（一般默认可能是2MB），这个栈主要用来保存函数递归调用时参数和局部变量。固定了栈的大小导致了两个问题：一是对于很多只需要很小的栈空间的线程来说是一个巨大的浪费，二是对于少数需要巨大栈空间的线程来说又面临栈溢出的风险。针对这两个问题的解决方案是：要么降低固定的栈大小，提升空间的利用率；要么增大栈的大小以允许更深的函数递归调用，但这两者是没法同时兼得的。相反，一个Goroutine会以一个很小的栈启动（可能是2KB或4KB），当遇到深度递归导致当前栈空间不足时，Goroutine会根据需要动态地伸缩栈的大小（主流实现中栈的最大值可达到1GB）。因为启动的代价很小，所以我们可以轻易地启动成千上万个Goroutine。

   Go的运行时还包含了其自己的调度器，这个调度器使用了一些技术手段，可以在n个操作系统线程上多工调度m个Goroutine。Go调度器的工作和内核的调度是相似的，但是这个调度器只关注单独的Go程序中的Goroutine。Goroutine采用的是半抢占式的协作调度，只有在当前Goroutine发生阻塞时才会导致调度；同时发生在用户态，调度器会根据具体函数只保存必要的寄存器，切换的代价要比系统线程低得多。运行时有一个runtime.GOMAXPROCS变量，用于控制当前运行正常非阻塞Goroutine的系统线程数目。

   在Go语言中启动一个Goroutine不仅和调用函数一样简单，而且Goroutine之间调度代价也很低，这些因素极大地促进了并发编程的流行和发展。

   所谓的原子操作就是并发编程中“最小的且不可并行化”的操作。通常，如果多个并发体对同一个共享资源进行的操作是原子的话，那么同一时刻最多只能有一个并发体对该资源进行操作。一般情况下，原子操作都是通过“互斥”访问来保证的，通常由特殊的CPU指令提供保护。当然，如果仅仅是想模拟下粗粒度的原子操作，我们可以借助于sync.Mutex来实现：
   ```go
   import (
       "sync"
   )

   var total struct {
       sync.Mutex
       value int
   }

   func worker(wg *sync.WaitGroup) {
       defer wg.Done()

       for i := 0; i <= 100; i++ {
           total.Lock()
           total.value += i
           total.Unlock()
       }
   }

   func main() {
       var wg sync.WaitGroup
       wg.Add(2)
       go worker(&wg)
       go worker(&wg)
       wg.Wait()

       fmt.Println(total.value)
   }
   ```

   用互斥锁来保护一个数值型的共享资源，麻烦且效率低下。标准库的sync/atomic包对原子操作提供了丰富的支持。我们可以重新实现上面的例子：
   ```go
   import (
       "sync"
       "sync/atomic"
   )

   var total uint64

   func worker(wg *sync.WaitGroup) {
       defer wg.Done()

       var i uint64
       for i = 0; i <= 100; i++ {
           atomic.AddUint64(&total, i)
       }
   }

   func main() {
       var wg sync.WaitGroup
       wg.Add(2)

       go worker(&wg)
       go worker(&wg)
       wg.Wait()
   }
   ```

   在Go语言中，同一个Goroutine线程内部，顺序一致性内存模型是得到保证的。但是不同的Goroutine之间，并不满足顺序一致性内存模型，需要通过明确定义的同步事件来作为同步的参考。如果两个事件不可排序，那么就说这两个事件是并发的。为了最大化并行，Go语言的编译器和处理器在不影响上述规定的前提下可能会对执行语句重新排序（CPU也会对一些指令进行乱序执行）。比如下面这个程序：
   ```go
   func main() {
       go println("你好, 世界")
   }
   ```
   根据Go语言规范，main函数退出时程序结束，不会等待任何后台线程。因为Goroutine的执行和main函数的返回事件是并发的，谁都有可能先发生，所以什么时候打印，能否打印都是未知的。

   解决问题的办法就是通过同步原语来给两个事件明确排序：
   ```go
   func main() {
       done := make(chan int)

       go func(){
           println("你好, 世界")
           done <- 1
       }()

       <-done
   }
   ```
   **无缓存的Channel上的发送操作总在对应的接收操作完成前发生**

### 常见的并发模式

   并发编程的核心概念是同步通信，但是同步的方式却有多种。我们先以大家熟悉的互斥量sync.Mutex来实现同步通信。
   ```go
   func main() {
       var mu sync.Mutex

       mu.Lock()
       go func(){
           fmt.Println("你好, 世界")
           mu.Unlock()
       }()

       mu.Lock()
   }
   ```

   使用sync.Mutex互斥锁同步是比较低级的做法。我们现在改用无缓存的管道来实现同步：
   ```go
   func main() {
       done := make(chan int)

       go func(){
           fmt.Println("你好, 世界")
           <-done
       }()

       done <- 1
   }
   ```

   基于select实现的管道的超时判断：
   ```go
   select {
   case v := <-in:
       fmt.Println(v)
   case <-time.After(time.Second):
       return // 超时
   }
   ```

   通过select的default分支实现非阻塞的管道发送或接收操作：
   ```go
   select {
   case v := <-in:
       fmt.Println(v)
   default:
       // 没有数据
   }
   ```

   我们通过select和default分支可以很容易实现一个Goroutine的退出控制：
   ```go
   func worker(cannel chan bool) {
       for {
           select {
           default:
               fmt.Println("hello")
               // 正常工作
           case <-cannel:
               // 退出
           }
       }
   }

   func main() {
       cannel := make(chan bool)
       go worker(cannel)

       time.Sleep(time.Second)
       cannel <- true
   }
   ```

   但是管道的发送操作和接收操作是一一对应的，如果要停止多个Goroutine那么可能需要创建同样数量的管道，这个代价太大了。其实我们可以通过close关闭一个管道来实现广播的效果，所有从关闭管道接收的操作均会收到一个零值和一个可选的失败标志。
   ```go
   func worker(cannel chan bool) {
       for {
           select {
           default:
               fmt.Println("hello")
               // 正常工作
           case <-cannel:
               // 退出
           }
       }
   }

   func main() {
       cancel := make(chan bool)

       for i := 0; i < 10; i++ {
           go worker(cancel)
       }

       time.Sleep(time.Second)
       close(cancel)
   }
   ```

   我们通过close来关闭cancel管道向多个Goroutine广播退出的指令。不过这个程序依然不够稳健：当每个Goroutine收到退出指令退出时一般会进行一定的清理工作，但是退出的清理工作并不能保证被完成，因为main线程并没有等待各个工作Goroutine退出工作完成的机制。我们可以结合sync.WaitGroup来改进：
   ```go
   func worker(wg *sync.WaitGroup, cannel chan bool) {
       defer wg.Done()

       for {
           select {
           default:
               fmt.Println("hello")
           case <-cannel:
               return
           }
       }
   }

   func main() {
       cancel := make(chan bool)

       var wg sync.WaitGroup
       for i := 0; i < 10; i++ {
           wg.Add(1)
           go worker(&wg, cancel)
       }

       time.Sleep(time.Second)
       close(cancel)
       wg.Wait()
   }
   ```

   在Go1.7发布时，标准库增加了一个context包，用来简化对于处理单个请求的多个Goroutine之间与请求域的数据、超时和退出等操作，官方有博文对此做了专门介绍。我们可以用context包来重新实现前面的线程安全退出或超时的控制:
   ```go
   func worker(ctx context.Context, wg *sync.WaitGroup) error {
       defer wg.Done()

       for {
           select {
           default:
               fmt.Println("hello")
           case <-ctx.Done():
               return ctx.Err()
           }
       }
   }

   func main() {
       ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)

       var wg sync.WaitGroup
       for i := 0; i < 10; i++ {
           wg.Add(1)
           go worker(ctx, &wg)
       }

       time.Sleep(time.Second)
       cancel()

       wg.Wait()
   }
   ```
   当main函数完成工作前，通过调用cancel()来通知后台Goroutine退出，这样就避免了Goroutine的泄漏。

### 错误和异常

   错误处理是每个编程语言都要考虑的一个重要话题。在Go语言的错误处理中，错误是软件包API和应用程序用户界面的一个重要组成部分。

   在程序中总有一部分函数总是要求必须能够成功的运行。比如strconv.Itoa将整数转换为字符串，从数组或切片中读写元素，从map读取已经存在的元素等。这类操作在运行时几乎不会失败，除非程序中有BUG，或遇到灾难性的、不可预料的情况，比如运行时的内存溢出。如果真的遇到真正异常情况，我们只要简单终止程序就可以了。

   排除异常的情况，如果程序运行失败仅被认为是几个预期的结果之一。对于那些将运行失败看作是预期结果的函数，它们会返回一个额外的返回值，通常是最后一个来传递错误信息。如果导致失败的原因只有一个，额外的返回值可以是一个布尔值，通常被命名为ok。比如，当从一个map查询一个结果时，可以通过额外的布尔值判断是否成功：
   ```go
   if v, ok := m["key"]; ok {
       return v
   }
   ```

   我们可以通过defer语句来确保每个被正常打开的文件都能被正常关闭：
   ```go
   func CopyFile(dstName, srcName string) (written int64, err error) {
       src, err := os.Open(srcName)
       if err != nil {
           return
       }
       defer src.Close()

       dst, err := os.Create(dstName)
       if err != nil {
           return
       }
       defer dst.Close()

       return io.Copy(dst, src)
   }
   ```
   defer语句可以让我们在打开文件时马上思考如何关闭文件。不管函数如何返回，文件关闭语句始终会被执行。同时defer语句可以保证，即使io.Copy发生了异常，文件依然可以安全地关闭。

   在处理错误返回值的时候，没有错误的返回值最好直接写为nil。
   ```go
   func returnsError() error {
       var p *MyError = nil
       if bad() {
           p = ErrBad
       }
       return p // Will always return a non-nil error.
   }

   // 上面错误的值是一个MyError类型的空指针，最好像下面这样直接写为nil
   func returnsError() error {
       if bad() {
           return (*MyError)(err)
       }
       return nil
   }
   ```

   Go语言函数调用的正常流程是函数执行返回语句返回结果，在这个流程中是没有异常的，因此在这个流程中执行recover异常捕获函数始终是返回nil。另一种是异常流程: 当函数调用panic抛出异常，函数将停止执行后续的普通语句，但是之前注册的defer函数调用仍然保证会被正常执行，然后再返回到调用者。在异常发生时，如果在defer中执行recover调用，它可以捕获触发panic时的参数，并且恢复到正常的执行流程。

   正确的recover函数使用:
   ```go
   func MyRecover() interface{} {
       return recover()
   }

   func main() {
       // 可以正常捕获异常
       defer MyRecover()
       panic(1)
   }

   // 这种方式也可以
   func main() {
       // 可以正常捕获异常
       defer func() {
            if r := recover(); r != nil {
                fmt.Println(r)
            }
        }()
       panic(1)
   }
   ```

   错误的recover函数使用:
   ```go
   func main() {
       if r := recover(); r != nil {
           log.Fatal(r)
       }

       panic(123)

       if r := recover(); r != nil {
           log.Fatal(r)
       }
   }

   // 这种也是错误的
   func main() {
       defer func() {
           // 无法捕获异常
           if r := MyRecover(); r != nil {
               fmt.Println(r)
           }
       }()
       panic(1)
   }

   func MyRecover() interface{} {
       log.Println("trace...")
       return recover()
   }

   // 这种也是错误的
   func main() {
       defer func() {
           defer func() {
               // 无法捕获异常
               if r := recover(); r != nil {
                   fmt.Println(r)
               }
           }()
       }()
       panic(1)
   }
   ```
   当然，为了避免recover调用者不能识别捕获到的异常, 应该避免用nil为参数抛出异常。
