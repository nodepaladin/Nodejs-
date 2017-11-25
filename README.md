# Thinking-in-Nodejs
## 1.Nodejs Eventloop，Timers and process.nextTick()
### 什么是Eventloop
Eventloop是允许Nodejs执行非阻塞I/O操作的核心架构，尽管事实上JavaScript是单线程-随时可能撂挑子给系统内核  
因为大部分现代内核都是多线程，他们可以在后台并行处理多个任务。当其中一个任务完成，内核就告诉Nodejs，将其对应的回调callback放入轮询队列（poll queue），这个回调将会在一定时机被执行。下面将会把这些问题展开讨论。
### Eventloop详解
当Nodejs启动的时候，会初始化Eventloop，处理提供的可能产生异步API调用、计划计时器，或者调用process.nextTick()的输入脚本，然后开始处理event loop  
下面这张图简单的展示了eventloop的操作顺序  
```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          |<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘

```
> Tips：每个方框可以看做是一个“阶段”  

每个“阶段”都有一个先进先出（FIFO）的回调队列要执行。而每个阶段都有其独特的方式，一般的，当event loop进入一个给定的阶段，他将会执行那个阶段定义的所有操作，然后执行那个阶段队列中的回调直到这个队列耗尽或者到达回调的最大个数。当队列耗尽或者回调执行个数到达限制，event loop将会移向下一个阶段，以此类推  
由于任何的这些操作可能会调度更多的操作和新事件被轮询阶段（poll phase）处理  
### <span id="what">阶段概览</span>
- timers 这个阶段执行被函数setTimeout（）和setInterval（）调度的回调  
- I/O callbacks 执行几乎所有除timer、close、setImmediate（）之外的回调  
- idle，prepare 只被内部使用  
- poll 获取新的I/O事件，Nodejs在这里可能会阻塞  
- check setImmediate（）的回调在这个阶段被调用  
- close callbacks 例如<code>socket.on（'close',...）</code>    
  
在event loop每次运行期间，Nodejs会检查他是否在等待一些异步I/O或者timer，如果没有（异步I/O或者timer），则完全关闭event loop。  
### 阶段详细介绍  
#### timers  
定时器在给定回调后边指定了回调可能执行的时间阈值，而不是某个人期望回调执行的准确时间。定时器回调将会尽可能早的在其定义后的指定时间执行，无论如何，操作系统调度任务或者其他回调都可能会使定时器回调延迟执行。 
> Tips：从技术上说，poll阶段控制着何时定时器回调执行  

举个例子，你定义了一个定时器回调在100ms后执行，同时一个其他的异步读文件操作（耗时95ms）开始执行。代码大致如下  
```
   const fs = require('fs');
   function someAsyncOperations(callback){
   //假设这读取文件耗时95ms
      fs.readFile('/data/file.js', callback);
   }
   //定时器开始时间
   const timeoutScheduled = Date.now();
   //定义定时器
   setTimeOut(() => {
      let diffTime = Date.now() - timeoutScheduled;
      console.log(`延迟执行时间为${diffTime}`);
   }, 100);
   someAsyncOperations(() => {
      let now = Date.now();
      while(Date.now - now < 10){
         //模拟执行时间10ms
      }
   });
```
当event loop进入poll阶段的时候，poll这个时候只有个空的队列（fs.readFile()还没有完成，因此回调还没有进入回调队列），event loop在最早的定时器时间阈值之前将会等待数毫秒。当过去95ms的时候，fs.readFile()完成了读取操作，他的回调（需耗时10ms）被添加到poll的执行队列中并被执行。经过10ms这个回调执行完毕，此时队列中没有其他的回调，所以event loop看到最早的timer定时器阈值到了后就绕回timer阶段去执行定时器的回调。这个代码示例中，定时器被定义到定时器回调被执行之间的时间是105ms。  
> 为了防止event loop在poll阶段饥饿，libuv（实现Node.js的event loop和所有平台异步行为的C语言库）在停止对更多事件轮询前设置了一个最大的轮询次数值（依赖系统）。  
#### I/O回调  
这个阶段执行一些系统回调诸如TCP类型的错误。例如，如果一个TCP的socket尝试连接的时候收到了<code>ECONNREFUSED</code>，一些\*nix系统想要报告错误。这将在I/O阶段顺序执行。  
#### poll  
**poll** 阶段主要有两类函数：  
- 执行已经到时的定时器脚本  
- 处理poll队列中的事件

当event loop进入poll阶段且没有计划定时器，将会发生如下两种情况之一：  
- 如果poll队列不为空，event loop将会循环访问poll队列并同步执行其中的回调函数直到遍历完或者遍历数到达系统上限。 
- 如果poll队列为空，则会发生下面两种情况之一：  
 1.  如果脚本是用setImmediate（）方法定义的，event loop将会结束poll阶段进入check阶段执行这些脚本。
 2.  如果没有用setImmediate（）定义脚本，event loop将会等待poll队列中被加入回调，然后立即执行。  
一旦poll队列为空，event loop将会检查已到达阈值时间的计时器。如果一个或更多计时器时间到达，event loop将会绕回timers阶段去执行这些计时器的回调。
#### check
这个阶段允许人在poll阶段完成后立即执行回调。如果poll阶段变得空闲并且脚本已经被setImmediate（）入列，event loop就可能执行check阶段而不是继续等待。
setImmediate（）实际上是一个运行在event loop独立阶段的特殊定时器，他在poll阶段完成后使用libuv的API调度回调执行。
通常的，当代码被执行，event loop最终会到达poll阶段等待连接、网络请求等的到来。无论如何，当setImmediate（）定义了回调函数，并且poll阶段变为idle的时候，event loop将会结束等待并去执行check阶段。  
#### close callbacks  
如果一个socket或者句柄突然被关闭（比如<code>socket.close()</code>），<code>close</code>事件将会在这个阶段被发出，否则将会被<code>process.nextTick()</code>  
#### setImmediate() VS setTimeout()
setImmediate()和setTimeout()非常像，但是调用的时机不同，他们的表现行为不同。
- setImmediate（）被设计用来在poll阶段完成后执行一段脚本  
- setTimeout()被设计用来在一段时间后执行一段脚本  
timers执行的顺序的变化取决于他们被调用时的环境。如果两者都在主要模块内部被调用，那么计时过程将受到进程的性能约束（会受到运行在机器上的其他程序影响）  
比如，如果我们在I/O循环外（也就是主模块）运行下面的脚本，两个计时器的执行顺序将是不确定的，那么执行顺序将会受到进程性能的约束。  
```
setTimeout(() => {
    console.log('timeout');
  }, 0);
  
  setImmediate(()=>{
    console.log('immediate');
  });
  
```
然而，如果将timers移到I/O循环内部，则setImmediate()方法的回调总是首先执行。  
```
const fs = require('fs');

fs.readFile('./hello', (error, data) => {
  console.log('data is '+data)
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  
  setImmediate(()=>{
    console.log('immediate');
  })
});

```

