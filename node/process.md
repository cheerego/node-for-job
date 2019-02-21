## Process 与 多进程

### 多进程
为了解决同步架构的并发问题，一个简单的改进是通过进程的复制来服务跟多的请求。这样每一个链接都需要一个进程来服务,即100个连接需要启动100个进程来服务，这是非常昂贵的。在进程复制的过程中，需要复制进程的内部状态，对于每个连接都进行这样复制的话，相同的状态将会在内存中存在很多份，造成浪费。并且这个过程中由于复制较多的数据，启动是较为缓慢的。
为了解决启动缓慢的问题，预复制被引入到服务模型，即预先复制一定数量的进程。同时将进程进行复用，避免进程创建和销毁带来开销。

### 多线程
为了解决多进程复制过程中的浪费，多线程被引入，让一个线程服务一个请求。线程相对于进程开销要小很多，并且线程的数据是共享的，内存的浪费可以得到解决，并且利用线程池可以减小创建和销毁的开销。但是多线程锁面临的并发问题只能说比多进程略好，因为每个线程都拥有独立的堆栈，这个堆栈都需要占用一点的内存空间。另外一个CPU在同一时刻只能做一件事，操作系统只能通过将CPU切分为时间片的方式，让线程较为均匀的使用CPU资源，但操作系统内核在切换线程的同时也要切换线程的上下文，当线程数量过多时，时间将会消耗在上下文切换中。

### 事件驱动
事件驱动(event-driven)是nodejs中的第二大特性。何为事件驱动呢？简单来说，就是通过监听事件的状态变化来做出相应的操作。比如读取一个文件，文件读取完毕，或者文件读取错误，那么就触发对应的状态，然后调用对应的回掉函数来进行处理。

### child_process
我们可以通过 child_process 这个模块来创建子进程。

* spawn:`启动一个子进程执行命令`
* exec:`启动一个子进程执行命令，与 spawn 不同的是，具有一个回调函数获知子进程的状况`
* execFile:`启动一个子进程执行可执行文件`
* fork:`与 spawn 类似，需要指定一个 JavaScript 文件模块`

#### Demo
```javascript
// work.js
console.log('123');

// index.js
let cp = require('child_process');

cp.spawn('node',['work.js']);
cp.exec('node work.js',function(err,stdout,stdderr) {
  
})

// 这里的 work.js 需要加上 #!/usr/bin/env node
cp.execFile('work.js',function(err,stdout,stdderr) {
  
})

cp.fork('./work.js')

// exec execFile fork 的底层都是调用 spawn

```

### 进程间通讯
`parent.js`:
```javascript
let cp = require('child_process');

let child = cp.fork('./child.js');

child.on('message', function (message) {
    console.log(`in parent,child say:${message}`);
});

child.send('parent:Do you have enough money?');

```

`child.js`:
```javascript
process.on('message', function (msg) {
    console.log('in child,parent say:'+msg);
});

process.send('Yes, I do');
```
`结果`：
```
in child,parent say:parent:Do you have enough money?
in parent,child say:Yes, I do
```

#### IPC
进程间通讯原理：通过 fork 或者其他 API 创建子进程，为了使父子进程进行通讯,父子进程之间将会创建IPC通道，通过IPC通道，父子进程通过send和message 进行通讯。
Linux 下 Domain Socket
Windows 下 named pipe

```
进程间通讯技术
命名管道
匿名管道
socket
信号量:使用 kill -l 查看所有信号量
KILL 杀死/删除进程，编号为9
INT 中断信号，编号为2,当用户输入Ctrl+C时由终端驱动程序发送INT信号INT信号是终止当前操作的请求，简单程序捕获到INT信号时应该退出，拥有命令行或者输入模式的那些,程序应该停止他们正在做的事情，清除状态，并等待用户再次输入。
QUIT
USR1

共享内存
消息队列
Domain Socket
```                                  



#### Cluster 模块
Cluster 原理，cluster 模块有 net 模块和 child_process.fork 实现，为何可以一个端口被多个进程监听，然后将master 进程创建的 fd 通过IPC传递个子进程。

```javascript
//parent.js
var cp = require('child_process');
var child1 = cp.fork('./child.js');
var child2 = cp.fork('./child.js');

var server = require('net').createServer();

server.on('connection',function(socker) {
  socker.end('handle by parent');
})

server.listen(1337,function() {
  child1.send('server',server)
  child2.send('server',server)
})

//child.js
process.on('message',function(m,server) {
  if (m=='server'){
      server.on('connection',function(socker) {
        socker.on('handle by child!')
      })
  } 
})
```
