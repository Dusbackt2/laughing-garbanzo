### stay foolish
muke go redis
 1.GMP模型
 2.gopark() 找可接管的阻塞
https://astexplorer.net/ 
dlv
go tool compile -S test.go 
go build ./test.go && go tool objdump ./test | grep -E 'test.go:7|test.go:9'
编译原理的书
https://interpreterbook.com/
https://github.com/motemen/gore

### 123::
```
__rt0_amd64    asm_amd64.s
  asm_amd64.s:16	0x1053ca0		488b3c24		MOVQ 0(SP), DI			 
  asm_amd64.s:17	0x1053ca4		488d742408		LEAQ 0x8(SP), SI		
  asm_amd64.s:18	0x1053ca9		e912000000		JMP runtime.rt0_go.abi0(SB)	
```
  复制argc,argv到栈上

  初始化g0
  runtime.g0<br>
  **g0是为了调度携程而产生的携程，第一个携程**

  runtime.check()<br>
  检查各种类型的长度,指针操作,结构体字段的偏移量
  atomic原子操作,CAS操作,栈的大小是否是2的幂次<br>
  
  runtime.args 参数的数量和值赋值给argc argv 拷贝到go语言中<br>

  runtime.osinit 判断核数<br>

  runtime.schedinit 调度器初始化<br>
  全局栈空间的分配，加载命令行参数到os.Args,堆内存空间初始化,加载操作系统环境变量<br>
  初始化当前系统线程,垃圾回收参数初始化,算法初始化(hash,map),设置process数量<br>

  runtime.mainPC 取runtime.main方法的地址，位于proc.go<br>
  runtime.newproc 启了一个新携程来运行runtime.main方法<br>
  runtime.mstart start is M <br>

  runtime.main 主要干了什么<br>
  doInit初始化 <br>
  gcenable() 打开了垃圾回收器<br>
  do用户包依赖的init方法<br>
  main.main 执行 (main_main linked)<br>


  本地文件替代
  go mod 文件追加  replace github.com.dust => local/dust<br>
  go vender <br>

### 4::
 空结构的地址都相同zerobase地址,节省内存 map channel<br>

 string  stringstruct{} 字符串<br>
 反射包里有一个 StringHeader<br>
 s:="abc你"<br>
 sh:=(*reflect.StringHeader)unsafe.Pointer(&s)<br>
 sh.Len byte数组长度 range自动解码<br>
 遍历走range a b c 你  否则len是字节 a,b,c n1,n2,n3===>n1,n2,n3=='你'<br>
 string(rune(s)[:3]) 取前三个<br>

 slice
 slice struct{
    array unsafe.Pointer
    len int
    cap int
 }
```go
// 查看汇编代码
 go build -gcflags -S test.go
```

```
         0x0018 00024 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) LEAQ    type:[3]int(SB), AX
        0x001f 00031 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) PCDATA  $1, $0
        0x001f 00031 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) NOP
        0x0020 00032 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) CALL    runtime.newobject(SB)
        0x0025 00037 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) MOVQ    $1, (AX)
        0x002c 00044 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) MOVQ    $2, 8(AX)
        0x0034 00052 (/Users/dingjiajun/Documents/ltcd/tesss/test.go:6) MOVQ    $3, 16(AX)
```
append len cap <br>
runtime.growslice() 扩容<br>
期望容量大于当前容量的2倍，就取期望容量<br>
小于1024 翻倍<br>
大于1024 +25%<br>
切片扩容时，并发不安全<br>

map
开放寻址法 拉链法
```go
runtime.hmap struct{}

make(map[string]int ,10)===>runtime.makemap
map[string]int{
  "1":2,
  "2":3
}
```
小于25个直接赋值<br>
大于key放到数组value放到，循环赋值<br>
map计算取值<br>
hash0 取出hash ,后面与B位数取值桶号，前八位tophash()取出桶里的哪个值<br>

map扩容 hashGrow<br>
runtime.mapassign()<br>
装载因子超过6.5（平均每个槽位置6.5个）<br>
使用太多溢出桶<br>
开始扩容，新建桶，oldbucket newbucket 改成扩容状态<br>

map 并发
sync.Map 

接口 隐式
```go
iface.go 
iface struct{
  tab *itab
  data unsafe.Pointer
}

itab struct{
  interfacetype
  type
  hash
  -
  func[]
}

// 类型断言 c.(Car)
// go build -gcflags -S test.go

func(t Trunk) Drive(){

}
// 会额外实现一个方法
func(t *Trunk) Drive(){

}

// 如果是
func(t *Trunk) Drive(){

}
// 不会而外实现
func(t Trunk) Drive(){

}
```
空接口
runtime.eface
作为万能入参，函数调用时候，会新生成一个空接口，再传参

nil 空接口，空结构体区别<br>
***nil 是pointer channel func interface map slice的zero value***
```go
type Student struct {
}
student := new(Student)
fmt.Printf("student 的数据类型为:%T,值为：%v\n", student, student)
fmt.Println("student == nill :", student == nil)

student 的数据类型为:*main.Student,值为：&{}
student == nill : false
```

内存对齐
对齐系数
```go
unsafe.Alignof()
```
变量的内存地址必须被内存系数所整除
```go
type Demo struct{
  bool
  int16
  string
  struct{} 需要填充
}=24

type Demo struct{
  bool
  struct{} 紧跟前一位
  int16
  string
}=24
```

### 5::
协程
```go
type g struct{
  stack 协程栈
  sched {
    sp 指向现在运行的栈帧
    pc 程序计数器
  }
  atomicstatus 协程状态
  goid 协程id
  stackguard0 =Oxfffffade 抢占标志
}

m struct{
  g0
  curg 现在正在运行哪个协程
  mOS 每种操作系统的额外信息
}
```

单线程循环
```
shcedule()->excute()->gogo()->业务方法()->goexit()
    ↑                                       ↓
    <---------------------------------------
```

```go
go stack{
  goexit()
  do1()
  do2()
}

g0 stack{
  shcedule()
  excute()
  gogo()
}

runtime.shecdule(){
  gp *g 将要工作的协程
  从runtime.next 本地队列，全局队列，其他P窃取 p开始起作用
  execute(gp,inheritTime)
}

exceute(){
  给gp的协程赋值
  gogo(&gp,shced)
}

gogo(){汇编实现，平台相关} 插入 goexit栈帧（执行完业务方法自动退到goexit()方法） 跳转到协程计数器进行执行

runtime.goexit(汇编)-->调用 runtime.goexit1()

runtime.goexit1(){
  macll(goexit0()) 切换 g0栈 记录信息调用 shcedule()
}


GMP调度模型 解决锁

p struct{
  m unitptr 原始指针 服务于m
  runq[] gunintptr 本地队列
  runnext gunitptr
}
```

切换执行 解决协程饥饿问题<br>
全局饥饿 %61<br>
runtime.gopack()挂起<br>
entersyscall()->系统调用()->exitsyscall()->进到g队列<br>
抢占式<br>
runtime.morestack(),调用其他方法前都会调用这个<br>
本意是检查协程栈是否有足够的空间<br>
监控协程超过10ms<br>
stackguard0 =Oxfffffade 回到schedule 基于写协作的抢占式调度<br>

为了防止
```go
for true{
  i++
}
```

基于信号的抢占式调度，linux的信号<br>
注册SIGURG信号处理函数<br>
GC工作时，向目标线程发送信号<br>
线程收到信号，触发调度<br>
doSinPreempt() ->asyncPreempt()<br>

协程太多会出现的问题<br>
文件打开数限制<br>
内存限制<br>
调度开销过大<br>

channel缓冲区，协程池（谨慎使用，二次池化）<br>

### 6::
锁
atomic 硬件LOCK锁<br>
```go
sync.Mutex{ //信号量锁
                             后面三位
  state int32   WaiterShift  Starving  Woken  Locked 
  sema  uint32
}

semaRoot{
  lock  mutex
  treap *sudog
  nwait uint32
}

sudog{
  g *g
  next *sudog
  prev *sudog
}
```
semaacquire sema>0 获取锁 sema==0直接入队列，unpark挂起<br>
semarelease sema+1 nwait 不等于0 dequeue 唤醒<br>

sync.Mutex<br>
lock 多个g 锁 ，lockSlow() runtime_SamacquireMutex() 未获取自旋后sema休眠等待==<br>+sudog队列里<br>
unlock         unlockSlow()  runtime_Samarelease <br>
锁饥饿问题解决<br>
协程等待锁时间超过10ms切换饥饿模式<br>
新进g 获取不到直接进sudog<br>
被唤醒的直接获取<br>
没有协程在sudog中继续等待，回到正常模式<br>

```go
sync.RWMutex{
  w
  writerSem
  readerSem
  readerCount
  readerWait
}
```

先加mutex写锁，若已经被加写锁就阻塞等待<br>
readerCount-=num,负值阻塞读锁获取<br>
计算要多少个读锁释放<br>
如果需要读协程释放，陷入writerSem<br>
来了读协程，进入readerSem，readerCount来一个加一个<br>
解锁<br>
readerCount+=num,允许读锁获取<br>
释放readerSem中等待的读协程<br>
解锁mutex<br>
```
加读锁 readerCount>0的话 readerCount+=1
      readerCount<0    readerCount+=1 readerSem排队
解读锁 readerCount>0 readerCount-=1
      readerCount<0 readerCount-=1 readerWait-=1 最后一个唤醒写线程
```

```go
WaitGroup{
  noCopy
  waiter 多少个在等
  counter 还有几个没做完
  sema 
}

wait(){
  counter=0 直接返回
  counter>0
  进入sema队列 
} 

add(){
  归零者
  唤醒所有sema
} 
```

sync.Once

锁拷贝 go vet main.go 

### 7::   <br>
```
channel
hchan{
  ----------缓存区-------------
  qcount
  dataqsiz
  buf
  elemsize
  ele,type 
  ----------------------------
  sendx
  recvx
  sendq{
    *g
    *sudog
    *sudog
  }
  recvq

  lock
}
```
```go
ch<-
runtime.chansend1()
```
直接发送，有接收协程在等待<br>
直接将数据拷贝到G的接收变量并唤醒<br>

放入缓存<br>
获取可存入的缓存地址,存入数据<br>

休眠等待<br>
把自己包装成sudog放入sendq队列，休眠并解锁<br>

<-ch<br>
runtime.chanrecv1() runtime.chanrecv2()<br>
有等待的G从G接收 （无缓存）<br>
有等待的G从缓存接收<br>
接收缓存<br>
阻塞接收 <br>

### 8::
TCP epoll
***
新建多路复用器 epoll_create()------------------------->netpollinit(){
  新建epoll
  新建一个pipe管道用于中断epoll
  将“管道有事件到达”事件注册在Epoll中
}
***
往多路复用器中插入需要监听的事件 epoll_ctl()------------->netpollopen(){<br>
  传入一个Socket的FD，和pollDesc指针<br>
  pollDesc指针是socket相关的详细信息<br>
  pollDesc中记录了哪个协程休眠在等待此Socket<br>
  将Socket的可读，可写，断开事件注册到Epoll中<br>
}<br>
***
查询发生了什么事件 epoll_wait()------------------------>netpoll(){<br>
  调用epoll_wait()查询有哪些事件发生<br>
  根据Socket的pollDesc信息，返回哪些协程可以唤醒<br>
}<br>
***
NetWolkerPoller<br>
NetWolkerPoller初始化<br>
```go
poll_runtime_pollServerInit(){
  只初始化一次
  netpollinit()
}

pollcache{
  lock
  first *pollDesc
}

pollDesc{ //runtime包对socket的详细的描述
  link *pollDesc
  fd  socket名字或者id
  rg pdReady | pdWait | G waiting for read | nil
  wg pdReady | pdWait | G waiting for write | nil
}


NetWolkerPoller新增监听的socket
poll_runtime_pollOpen(){ link runtime_pollOpen()
  //socket名字或者id  
  pollcache.alloc() 在pollcache链表中分配一个pollDesc
  初始化 pollDesc rg,wg 0
}
```

NetWolkerPoller收发数据
收发数据分为两个场景:
***
```
  1.协程需要收发数据时，Socket已经可读写{
   g0协程 gcStart(){
      netpoll(){
        epollwait()
        r | w
        netpollready(){
          netpollunblock(){
            发现Socket可读或者可写，给rg｜wg置为pdReady(1)
          }
        }
      }
    }
  }
  协程调用 poll_runtime_pollWait(){
    判断pdReady
  }
  ```
***
  2.协程需要收发数据时，Socket暂时无法读写<br>
  ```
  g0协程 gcStart(){
    netpoll(){}
  }

  协程调用 poll_runtime_pollWait(){
    //发现对应的rg｜wg为0
    gopark(){
      给对应的 rg｜wg置为协程地址
      休眠等待
    }
  }
  ```
  runtime 循环调用netpoll()方法<br>
  发现Socket可读写时，给对应的查看对应的rg|wg<br>
  若为协程地址则返回协程地址<br>
  调度器开始调度对应协程<br>
***
net包<br>
go原生的网络包，实现了TCP,UDP，HTTP等网络操作<br>
建立链接 <br>
***
  ```
netListen(tcp,":8888"){
  ListenConfig
  listen(){
    resolveAddrList(){
      判断合法
    }
    listenTCP(){
      internetSocket(){
        sysSocket()
        newFD()
        listenStream(){
          Bind()
          fd.init(){
            fd.pd.init(){
              runtime_pollOpen()
            }
          }
        }
      }
      Listen状态的listener
    }
  }
}
  ```
  ***
新建socket,并执行bind<br>
新建一个FD(net包对Socket详情的描述)<br>
返回一个TCPListener对象<br>
将TCPListener的FD信息加入监听<br>
TCPListener对象本质上是一个LISTEN状态的Socket<br>
***
```
tcpListener.Accept(){
  accept(){
    fd.accept(){
      fd.pfd.accept(){
        switch(){
          syscall.EAGAIN:
          fd.pd.waitRead()-->poll_runtime_pollWait()
        } 
      }
    }.var返回新的链接
    newFD()
  }.var 返回fd
  newTCPConn
}
```
***
直接调用Socket的accept()<br>
如果失败，休眠等待新的链接<br>
将新的Socket包装为TCPConn变量返回<br>
将TCPConn的FD加入监听<br>
TCPConn本质上是一个ESTABLISHED状态的Socket<br>
***
```
conn.Read(){
  fd.Read(){
    fd.pfd.Read(){
      ignoringEINTRIO(syscall.Read)
      ? 有数据返回，没有数据fd.pd.waitRead() r模式 注册到epoll等到有数据再唤醒
    }
  }
}

conn.Write(){
  fd.Write(){
    fd.pfd.Write(){
      ignoringEINTRIO(syscall.Write)
      成功返回否则fd.pd.waitRead() w模式 注册到epoll等到有数据再唤醒
    }
  }
}
```

直接调用Socket原生读写方法<br>
如果失败，休眠等待<br>
被唤醒后调用系统Socket<br>

goroutine-per-connection 编程风格<br>
结合多路复用的性能和阻塞模型的简洁<br>

***
内存::
栈内存
协程栈的作用:
协程的执行路径
局部变量
函数传参
协程栈在堆内存上
堆内存在操作系统的虚拟内存上

```go

func sum(a,b) int{
  sum:=0
  sum=a+b
  return sum
}


func main(){
  a=3
  b=5
  print(
    sum(a,b)
  )
}
```
```
                             stack 初始 2K-4K          
-----------------------------------------------------------------------
runtime.main                                   
-----------------------------------------------------------------------
main.main栈帧                runtime.main的栈基值
                            a=3
                            b=5
                            sum函数的返回值======================>sum=8 
                            sum参数5
                            sum参数3
                            sum返回后的指令
-----------------------------------------------------------------------
                            main.main的栈基值
sum的栈帧                    sum=0----------------------->sum=8  回收栈帧
-----------------------------------------------------------------------
```

协程栈不够？变量多,大，栈帧调用多

逃逸分析
指针逃逸
```go
func a() *int{
  c:=0
  return &c
}
```
空接口逃逸(往往使用反射)

```go
func b(){
  c:=0
  fmt.Println(c)
}
```

大变量逃逸 超过64K
```go
make(int[],10000)
```

栈扩容
morestack() 判断栈空间是否足够
1.13之前分段栈
缺点：不连续空间来回跳转
连续栈
扩容+copy 扩缩容

堆内存
heapArena 
Go每次申请的虚拟内存单元是64MB
最多有20^20个内存单元
内存单元也叫heapArena
mheap=heapArena的集合

heap struct{
  mcentral
}

type heapArena struct{

}

线性分配
链表分配
分级分配
mspan  67种mspan

mcentral 目录索引
***
```
      class0                   class1                   class2
mcentral   mcentral     mcentral     mcentral    mcentral     mcentral
 scan       noscan        scan       noscan        scan       noscan
                                  ↘️            ↙️
 heapArena   heapArena   heapArena   heapArena  heapArena  heapArena
```
***
mcentral 互斥锁保护
为了高并发的性能
给每个P mchache
***
```
      p                 P1
  mchache            mchache
span*68+span*68    span*68+span*68
scan    noscan     scan    noscan
refill 交换操作 nextFreeFast
```
***

对象分级
Tiny 微对象(0,16B) 无指针<br>
Small 小对象[16B,32KB] <br>
Large 大对象(32KB,+00)<br>
***
mallocgc()
Tiny 分配==>拿到class2的，将多个微对象合并成一个16Byte存入<br>
Large makeSpanClass(0,noscan)
***
回收内存
GC
标记清除
标记整理
标记复制

什么不能被清除
栈上的指针，全局指针，寄存器中的指针 RootSet
可达性分析
stop the world

减小GC对性能的影响
***
三色标记，并发标记
黑色 灰色 白色
有用 有用 无用
已析 未分析
***
Yuasa删除屏障
删除指针的对象置灰
Dijkstra插入屏障
并发标记（插入）
黑色指向白色对象
白色对象置灰
***
GC触发
定时触发 sysmon定时检查
用户显示出发 runtime.gc()
申请内存的时候触发
***

高级特性::

cgo

```go
package main

/*
int sum(int a,int b){
  return a+b;
}
*/
import "C"

func main(){
  println(C.sum(1,1))
}
```

go tool cgo main.go
调度器和协程配合

defer
记录信息
函数结尾调用 deferreturn

在堆上开辟一个 p 里的 sched.deferpool
遇到defer语句将信息放入deferpool

栈上分配

recover
panic.go

反射
获取对象的类型
对any赋值
调用任意方法

元数据
reflect.Type
reflect.Value

```go
func main(){
  s:="mooky"
  sg:=reflect.TypeOf(s)
  sg.Name()
  sv:=reflect.ValueOf(s)
  sv
  sv.Interface().(string).var

}
```

```go
package main

import (
	"fmt"
	"reflect"
)

func CallAdd(f func(a, b int) int, x, y int) {
	v := reflect.ValueOf(f)
	if v.Kind() != reflect.Func {
		return
	}

	argv := make([]reflect.Value, 2)
	argv[0] = reflect.ValueOf(x)
	argv[1] = reflect.ValueOf(y)
	v2 := v.Call(argv)
	fmt.Println(v2[0].Int())
}

func Add(a, b int) int {
	c := a + b
	return c
}

func Add2(a, b int) int {
	c := a - b
	return c
}

func main() {
	CallAdd(Add, 2, 3)
}

```




























































