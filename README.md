# Thinking-in-Nodejs
### 1.Nodejs Eventloop，Timers and process.nextTick()
#### 什么是Eventloop
Eventloop是允许Nodejs执行非阻塞I/O操作的核心架构，尽管事实上JavaScript是单线程-随时可能撂挑子给系统内核  
因为大部分现代内核都是多线程，他们可以在后台并行处理多个任务。当其中一个任务完成，内核就告诉Nodejs，将其对应的回调callback放入轮询队列（poll queue），这个回调将会在一定时机被执行。下面将会把这些问题展开讨论。
#### Eventloop详解
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
由于任何的这些操作可能会编排更多的操作和新事件被轮询阶段（poll phase）处理  
#### 阶段概览  
- timers 这个阶段执行被函数setTimeout（）和setInterval（）编排的回调  
- I/O callbacks 执行几乎所有timer、close、setImmediate（）异常  
- idle，prepare 只被内部使用  
- poll 检索新IO事件，Nodejs在这里可能会阻塞  
- check setImmediate（）的回调在这个阶段被调用  
- close callbacks 例如<code>socket.on（'close',...）</code>    


**执行队列:** node中代码执行队列，非阻塞   
**任务队列:** 每当有异步任务执行完成就会将回调方法放入任务队列，当执行队列为空的时候，会执行执行队列中的回调事件  
**setTimeout(fn, xxx):** 将fn放到任务队列尾部  
**setTimeout(fn, 0):** 将放到任务对列首部  
**nextTick(fn):** 将fn放到执行队列尾部  
**setImmediate():** 和setTimeout(fn, 0)类似，将fn放到任务队列头部  
在event loop每次运行期间，Nodejs会检查他是否在等待一些异步I/O或者timer，如果没有（异步I/O或者timer），则完全关闭event loop。  
#### 阶段详细介绍  
#### timers  
定时器在给定回调后边指定了回调可能执行的时间阈值，而不是某个人期望回调执行的准确时间。定时器回调将会尽可能早的在其定义后的指定时间执行，无论如何，操作系统计划任务或者其他回调都可能会使定时器回调延迟执行。 
> Tips：从技术上说，poll阶段控制着何时定时器回调执行  
举个例子，你定义了一个定时器回调在100ms后执行，同时一个其他的异步读文件操作（耗时95ms）开始执行。代码大致如下  
<code>
   const fs = require('fs');
   
   function someAsyncOperations(callback){
      fs.readFile('/data/file.js', callback);
   }
   
   const timeoutScheduled = Date.now();
   
   setTimeOut(() => {
      let diffTime = Date.now() - timeoutScheduled;
      console.log(`延迟执行时间为${diffTime}`);
   }, 100);
   
   someAsyncOperations(() => {
      let now = Date.now();
      while(Date.now - now < 10){
 //模拟回调执行时间
      }
   })
</code>

