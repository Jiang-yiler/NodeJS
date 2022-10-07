## Node.js的架构
作为一个服务端框架，`Node.js`在运行时有许多依赖，其中最重要的两个是`V8`引擎和`libuv`库
- **`V8`引擎**使得`Node.js`能够运行`JavaScript`代码，是用`JavaScript`和`C++`开发的
- **`libuv`库**使得`Node.js`能够进行文件操作、网络操作等，是用`C++`开发的。其内部实现了事件循环和线程池：
    - **事件循环**负责处理简单的任务，比如执行回调函数、网络`IO`等
    - **线程池**负责处理更加复杂的任务，比如文件访问、压缩等

- 其它依赖
    - `http-parser`：用于解析`http`请求和响应
    - `c-ares`：异步`DNS`解析库，可以和事件循环统一起来，实现`DNS`的非阻塞异步解析
    - `crypto(OpenSSL)`：用于实现安全通信，加密解密
    - `zlib`：用于压缩和解压缩


## Node 进程，线程和线程池
当我们运行`Node.js`时，计算机后台便会开启一个`Node.js`的进程（`Node.js`本身也提供了用于进程管理的API）

与此同时，`Node.js`的运行是单线程的，也就是说不管有多少用户在访问应用程序，所有指令都在一个线程中执行，这使得它非常容易被堵塞。具体来说，当`Node.js`被启动时，会在单线程中依次执行以下操作：

**初始化项目👉执行顶层代码（不在回调函数中）👉加载模块👉注册回调函数👉开启事件循环（回调函数中）**

- 其中，事件循环扛起了一片天，会执行程序中的大部分任务，但有些任务确实过于复杂，如果在事件循环中执行，就会阻塞整个线程。time for 线程池
- 线程池提供了4个与主线程完全分开的线程，事件循环在遇到诸如**文件、密码、压缩、DNS查询等复杂操作**时，就会将这些任务交给线程池

## 事件循环
### 文

如果要用一句话概括事件循环的作用，按我的理解就是**事件循环会接收事件、执行所有在回调函数中的代码，并将复杂的任务交付给线程池**

事件循环有多个阶段，每个阶段都有自己的一个回调函数队列，其中**四个重要的阶段**：
1. 普通计时器阶段：`setTimeout()`等
2. `I/O`任务阶段：`http()`，`fs()`等文件和网络处理相关
3. 特殊计时器阶段：`setImmediate()`
4. `close`阶段：如关闭`Web Server`、`WebSocket`时触发的回调函数

以及**两个特殊的回调队列**：
1. `process.nextTick()`的回调：需要在当前事件循环结束之后**立即执行**某个特定回调时使用，类似`setImmediate()`，不同的是`setImmediate()`是在文件和网络处理之后执行
2. 其它微任务的回调（Resolved promises）

**上面的序号即表示执行的优先级**

**重要阶段的回调**，以`setTimeout()`为例，当事件循环到了普通定时器阶段，如果此时计时结束了，会立即执行`setTimeout()`中的回调函数；如果没有结束，事件循环会继续到下一个阶段，而且在这次`circle`中不会再执行这个定时器中的回调，就算在这期间计时结束了。直到下次进入定时器阶段再执行。

而那两个**特殊的回调队列**，以`process.nextTick()`为例，虽然名称叫`nextTick`，但`process.nextTick()`中的回调并不会在下一个`tick`中执行，而是在当前阶段（`parse`）结束后立即执行，也就是说，假如事件循环执行到了`I/O`事件阶段，此时有`process.nextTick()`中的回调需要执行，那么在执行完当前`I/O`事件的回调之后便会立即执行`process.nextTick()`中的回调，之后再执行特殊计时器的回调。

那么`Node.js`是如何判断事件循环的一个`circle`是否结束的？在执行完`close`阶段的回调函数后，Node.js会检查是否有还在计时的定时器或`I/O`任务，如果还有那就进行事件循环的下一轮`circle`；如果没有就直接退出事件循环，也就退出`Node.js`的程序了

### 图
为了更加直观，我将上面叙述的**从`Node.js`程序启动到退出**的整个过程依照个人的理解绘制成了流程图：


![demo-事件循环.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/903bbedec28c4c6da5e612bac44ce316~tplv-k3u1fbpfcp-watermark.image?)


### 代码
**初来乍到**，先看一段代码热热身


```javascript
const fs = require("fs");

setTimeout(() => {console.log("Timer 1 finished"), 0;});

setImmediate(() => {console.log("Immediate 1 finished"); });

fs.readFile("test-file.text", () => {console.log("I/O finished"); });

console.log("Hello from the top-level code"); 

```
以上代码输出结果：
```
// node 10.15.2
Hello from the top-level code
Timer 1 finished
Immediate 1 finished
I/O finished
```

有的人可能运行的结果是
```
// node 10.15.2
Hello from the top-level code
Timer 1 finished
Immediate 1 finished
I/O finished
```

究其原因，需要补充一个知识点：**`Node.js`会把`setTimeout(fn, 0)`强制改为`setTimeout(fn, 1)`**，[官方文档](https://nodejs.org/api/timers.html#settimeoutcallback-delay-args)有介绍，这是源码决定的。所以关键就在这个`1ms`秒，如果同步代码执行时间较长，进入`普通定时器`阶段时`1ms`已经过了，那么`setTimeout`执行，否则就先执行了`setImmediate`。每次我们运行脚本时，机器状态可能不一样，导致运行时有`1ms`的差距

下面再**小试牛刀**一下

```javascript
const fs = require("fs");

setTimeout(() => {console.log("Timer 1 finished"), 0;});

setImmediate(() => {console.log("Immediate 1 finished");});

fs.readFile("test-file.text", () => {
  console.log("I/O finished"); 
  console.log("-------"); // 方便直观的区分
  setTimeout(() => {console.log("Timer 2 finished"), 0;});
  setTimeout(() => { console.log("Timer 3 finished"), 3000;});
  setImmediate(() => {console.log("Immediate 2 finished");});
  process.nextTick(() => { console.log("Process.nextTick");});
});

console.log("Hello from the top-level code"); // 同步代码

```
与第一段代码相比，在`fs.readFile`中增加了三个定时器和一个`process.nextTick`，输入结果如下：
```
Hello from the top-level code
Timer 1 finished
Immediate 1 finished
I/O finished
-------
Process.nextTick
Immediate 2 finished
Timer 2 finished
Timer 3 finished
```
`---`上方的输出与之前的一样，在进入`fs.readFile`的回调后遇到`process.nextTick`会立马执行，之后再继续执行其它内容，在进入`定时器阶段`时，`Timer 3`还处于`pending`的状态，因此只能到下一个循环中执行，至于 `setImmediate`先于`setTimeout`执行，是因为在轮询时发现有`setImmediate`就会执行`setImmediate`中的回调


## 总结
- 事件循环会执行所有在回调函数中的代码，并将复杂的任务交付给线程池
- 在同一个异步回调中，`setImmediate`总是比`setTimeout(fn, 0)`先执行；如果都在顶层代码或者`setImmediate`回调里，执行的顺序取决于当时机器的状况
- `process.nextTick`和`setImmediate`的名字应该互换一下，这样才与它们实际的执行机制相符，因为`setImmediate`实际上在下一个循环执行，`nextTick`实际上是马上执行🤔🤔🤔


## 后续
需要说明的是，本文中所有代码运行环境的`node 10.15.3`，`node 11`之后，其事件循环的原理渐渐向浏览器中的事件循环靠拢，比如重要阶段的划分、微任务的执行等。因此本文目前只是一个引子，后续打算分三个阶段再对文章作持续补充：

1. `node 11` 前后事件循环原理的变化以及最新的标准，比如`pharse`的划分和`process.nextTick`的执行顺序
2. 浏览器中的`JavaSsript`引擎线程和事件循环原理
3. 事件循环案例实践



参考资料：

[The Node.js Event Loop, Timers, and process.nextTick() | Node.js (nodejs.org)](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

[Node.js event loop workflow & lifecycle in low level (voidcanvas.com)](https://www.voidcanvas.com/nodejs-event-loop/)

[Node 事件循环机制 - 掘金 (juejin.cn)](https://juejin.cn/post/6844904137662922760)

[剖析nodejs的事件循环 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903621444763662#heading-5)