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
> Tips：每个方框可以看做是一个“组”  

**执行队列:** node中代码执行队列，非阻塞 
**任务队列:** 每当有异步任务执行完成就会将回调方法放入任务队列，当执行队列为空的时候，会执行执行队列中的回调事件  
**setTimeout(fn, xxx):** 将fn放到任务队列尾部  
**setTimeout(fn, 0):** 将放到任务对列首部  
**nextTick(fn):** 将fn放到执行队列尾部  
**setImmediate():** 和setTimeout(fn, 0)类似，将fn放到任务队列头部  
