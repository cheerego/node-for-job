## Process 与 多进程

### 多进程
为了解决同步架构的并发问题，一个简单的改进是通过进程的复制来服务跟多的请求。这样每一个链接都需要一个进程来服务,即100个连接需要启动100个进程来服务，这是非常昂贵的。在进程复制的过程中，需要复制进程的内部状态，对于每个连接都进行这样复制的话，相同的状态将会在内存中存在很多份，造成浪费。并且这个过程中由于复制较多的数据，启动是较为缓慢的。
为了解决启动缓慢的问题，预复制被引入到服务模型，即预先复制一定数量的进程。同时将进程进行复用，避免进程创建和销毁带来开销。

### 多线程
为了解决多进程复制过程中的浪费，多线程被引入，让一个线程服务一个请求。线程相对于进程开销要小很多，并且线程的数据是共享的，内存的浪费可以得到解决，并且利用线程池可以减小创建和销毁的开销。但是多线程锁面临的并发问题只能说比多进程略好，因为每个线程都拥有独立的堆栈，这个堆栈都需要占用一点的内存空间。另外一个CPU在同一时刻只能做一件事，操作系统只能通过将CPU切分为时间片的方式，让线程较为均匀的使用CPU资源，但操作系统内核在切换线程的同时也要切换线程的上下文，当线程数量过多时，时间将会消耗在上下文切换中。

### 事件驱动
1234


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
进程间通讯原理：通过 fork 或者其他 API 创建子进程，为了使父子进程进行通讯
eereeee                                                     
