## 理解 Event Loop

参考文档:

[The Node.js Event Loop, Timers, and process.nextTick() 官方文档](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

[Node.js的事件循环(Event Loop)、Timer和process.nextTick()[翻译]](https://zhuanlan.zhihu.com/p/34451546)

[Node.js Event Loop 的理解 Timers，process.nextTick()](https://cnodejs.org/topic/57d68794cb6f605d360105bf)

[不要混淆nodejs和浏览器中的event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)

[翻译了一个关于事件循环的系列博文，请大家多多指教](https://cnodejs.org/topic/5c6ab1b6ed5543510be8cbe0#5c6d227ee1a81129a7ad8cd6)
#### 牛刀小试
```javascript
setTimeout(function() {
  console.log(1)
}, 0);
new Promise(function executor(resolve) {
  console.log(2);
  for( var i=0 ; i<10000 ; i++ ) {
    i == 9999 && resolve();
  }
  console.log(3);
}).then(function() {
  console.log(4);
});
console.log(5);
```
`2 3 5 4 1`


#### 什么是事件循环?
事件循环允许 Node.js 执行非阻塞 I/O 操作 - 尽管JavaScript是单线程的 - 通过尽可能将操作让系统内核执行。

由于大多数现代内核都是多线程的，因此它们可以处理在后台执行的多个操作。当其中一个操作完成时，内核会告诉Node.js，以便可以将相应的回调添加到轮询队列中以最终执行。我们将在本主题后面进一步详细解释。
 
#### Event Loop 概述

下图显示了事件循环操作顺序的简要概述。
```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     pending callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘

```
* **timers**:此阶段执行由setTimeout() 和调度的回调setInterval()    
* **pending callbacks**:执行除了 close事件的callbacks、被timers(定时器，setTimeout、setInterval等)设定的callbacks、setImmediate()设定的callbacks之外的callbacks
* **idle, prepare**:仅在内部使用
* **poll**:获取新的I/O事件, 适当的条件下node将阻塞在这里
* **check**:setImmediate()在这里调用回调 
* **close callbacks**:一些关闭回调，例如socket.on('close', ...)


##### 事件循环示例

###### 1.poll 阶段会阻塞
```
var fs = require('fs');

function someAsyncOperation (callback) {
  // 花费2毫秒
  fs.readFile(__dirname + '/' + __filename, callback);
}

var timeoutScheduled = Date.now();
var fileReadTime = 0;

setTimeout(function () {
  var delay = Date.now() - timeoutScheduled;
  console.log('setTimeout: ' + (delay) + "ms have passed since I was scheduled");
  console.log('fileReaderTime',fileReadtime - timeoutScheduled);
}, 10);

someAsyncOperation(function () {
  fileReadtime = Date.now();
  while(Date.now() - fileReadtime < 20) {

  }
});
```
**定时器执行 20 ms 以上**
* 读取一次文件的操作一般在 3 ms 左右
* 进入初始化时间循环
* timer 阶段,无 callback 到达,设置的是10ms
* 进入 poll, poll 阻塞,2ms 拿到数据,将 callback 加入到 poll queue 中,执行 callback 花费 20 ms, poll 处于空闲状态,但是设置的timer发现超过了该值,立即循环回到timer阶段执行 callback
* 所以最后 setTimeout 实际输出的时间是 22 ms


###### 2.poll 阶段不阻塞
```
let start = new Date();
setTimeout(() => {
    console.log('timer:'+(new Date() - start));
},10);

// 花费35ms
http.get('http://www.baidu.com', function (res) {
    console.log('http:'+(new Date() - start));
});
//
timer:13
http:38

```

**定时器执行 10 ms 以上**
* 本机http.get 访问百度 35 ms
* 进入初始化时间循环
* timer 阶段,无 callback 到达,设置的是10ms
* 进入 poll, poll 阻塞,当运行了10ms是依然空闲,但设置的timer已到达时间,event loop 需要循环到 timer 阶段,然后执行完timer的callback,
然后继续回到poll阶段阻塞等待io执行完成,当在35ms的时候异步IO完成,将其的callback加入到poll queue中



###### 3.poll 阶段不阻塞
```
let start = new Date();
setTimeout(() => {
    console.log('timer:'+(new Date() - start));
},10);


fs.readFile(__filename, () => {
    console.log('fs:'+(new Date() - start));
});

setImmediate(() => {
    console.log('check:'+(new Date() - start))
});
//
check:3
fs:9
timer:11

```
* 没有timer
* 进入poll 阻塞,发现有 setImmediate ,立马进入check
* 没有timer,继续回到 poll 阻塞,然后文件读取完成,callback加入poll queue,执行callback,然后继续阻塞,发现有了timer
* 执行 10 ms 的timer


 
##### setTimeout 与 setImmediate
setImmediate并且setTimeout()是相似的，但根据它们被调用的时间以不同的方式表现。

* setImmediate()设计用于在当前poll阶段完成后的check阶段执行脚本 。
* setTimeout() 安排在经过最小阈值（ms）后运行的脚本。

```
setTimeout(() => console.log(1));
setImmediate(() => console.log(2));
```
上面这段代码的执行顺序是不确定的。

这是因为setTimeout的第二个参数默认为0。但是实际上，Node 做不到0毫秒，最少也需要1毫秒，根据官方文档，第二个参数的取值范围在1毫秒到2147483647毫秒之间。也就是说，setTimeout(f, 0)等同于setTimeout(f, 1)。

实际执行的时候，进入事件循环以后，有可能到了1毫秒，也可能还没到1毫秒，取决于系统当时的状况。如果没到1毫秒，那么 timers 阶段就会跳过，进入 check 阶段，先执行setImmediate的回调函数。


#### process.nextTick()
您可能已经注意到，process.nextTick()虽然它是异步API的一部分，但未在图中显示。这是因为 process.nextTick()从技术上讲，它不是事件循环的一部分。相反，nextTickQueue无论事件循环的当前阶段如何，都将在当前操作完成后处理。
process.nextTick()不在event loop的任何阶段执行，而是在各个阶段切换的中间执行,即从一个阶段切换到下个阶段前执行。

```javascript
var fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('setTimeout');
  }, 0);
  setImmediate(() => {
    console.log('setImmediate');
    process.nextTick(()=>{
      console.log('nextTick3');
    })
  });
  process.nextTick(()=>{
    console.log('nextTick1');
  })
  process.nextTick(()=>{
    console.log('nextTick2');
  })
});
//
node eventloop.js
nextTick1
nextTick2
setImmediate
nextTick3
setTimeout
```

从poll —> check阶段，先执行process.nextTick，
nextTick1
nextTick2
然后进入check,setImmediate，
setImmediate
执行完setImmediate后，出check,进入close callback前，执行process.nextTick
nextTick3
最后进入timer执行setTimeout
setTimeout
process.nextTick()是node早期版本无setImmediate时的产物，node作者推荐我们尽量使用setImmediate。
