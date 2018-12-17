## Cluster 使用与源码解析

* [负载均衡](#负载均衡)
* 只监听一个端口
* 平滑重启
* work 的分发机制
* 进行通讯
* 源码解析


### 负载均衡
`单线程写法`
```javascript
const http = require('http');
const pid = process.pid;
http.createServer((req, res) => {
    res.end(`hello world!${pid}`);
}).listen(8080,()=>{
    console.log('listening at 8080');
});
```


`多线程写法`
```javascript
const http = require('http');
const cluster = require('cluster');
const os = require('os');
if (cluster.isMaster) {
    const cpus = os.cpus().length;
    console.log('forking for ', cpus, ' CPUS');
    for (let i = 0; i < cpus; i++) {
        cluster.fork();
    }
} else {
    http.createServer((req, res) => {
        res.end(`hello world!`);
    }).listen(8080,()=>{
        console.log('listening at 8080');
    });
}
```



