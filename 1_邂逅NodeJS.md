在学习`Node.js`时，发现一处课程通过介绍网站开发来引入`Node.js`的思路很有意思，在此将整个过程以自己的理解记录了下来，供大家学习交流😀😀😀
## 前端和后端
众所周知，前端程序yuan通常写的是**浏览器**相关的bug，后端程序yuan写的则是**服务器**和**数据库**相关的bug👨‍🦳

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b0e7447e3e14b2c9f8b2b295dae8615~tplv-k3u1fbpfcp-watermark.image?)

上面提到的服务器，同样众所周知，可以简单理解为一台连网的**提供某种特定服务**的电脑，所以对于`Web`服务器来说，它具备的功能如下：
- 首先，存储文件的功能，比如`.html`,`.css`,`.js`文件
- 其次，与浏览器通信的功能，能接收浏览器发送的请求，并提供响应，这个功能由`HTTP Server`提供
- 以上两个功能就构成了静态服务器，为浏览器提供**静态资源**，如果想要提供**动态资源**，就需要用到数据库和`App Server`

## 静态网站

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4cdf68bb130432f9411f2bd3ff21788~tplv-k3u1fbpfcp-watermark.image?)

对静态网站来说，最初上传的时候网站是什么样，此后就一直是什么样，不会发生任何变化。

`Web`服务器此时就是一个静态服务器，它将网站渲染需要的文件直接发送给浏览器，而浏览器也原封不动地将这些文件渲染出来。
    

## 动态网站


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2132bc1d6ce4cf28d733a4eaa55dd46~tplv-k3u1fbpfcp-watermark.image?)

对动态网站来说，浏览器会根据用户的操作或者数据库的变化随时更改页面展示的内容。

动态网站渲染文件的获取通常需要**数据库**以及从数据库获取数据的**应用程序**，每当从浏览器发送一个请求到服务器时，应用程序会先从数据库中获取需要的数据，然后服务器会将数据丢到模板（`jsp`，`php`等）中生成`.html`，`.css`，`.js`文件，然后返回给浏览器。以上过程也被称为**服务端渲染**，因为渲染页面所需要的文件在服务端就组装好了。

但上述的服务端渲染存在几个问题。首先，代码会变得非常复杂，需要兼容各种运行情况，而且逻辑混杂，难以维护；其次，每次页面需要有变化，都得重新获取数据，然后再基于模板去组装；再有，对前端开发人员不太友好，如果想要开发页面，需要去了解`PHP`和`Java`等前端开发本身用不到的语言

于是，客户端渲染应运而生：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1457fb9654d74ca092b9cc67f1d0d92a~tplv-k3u1fbpfcp-watermark.image?)

在**客户端渲染**的模式中，服务器只提供`API`(Application Programming Interface，即应用程序接口)来返回数据，通常是`JSON`格式，而不是像服务端渲染那样返回需要展示的完整页面，浏览器接收到数据之后，同样会通过模板来生成页面，通常是使用`Vue`、`React`之类的前端框架，渲染页面的过程就来到了客户端。

但是与服务端渲染相比，客户端渲染有两个突出的缺点：一个是不利于`SEO`（Search Engine Optimazition，即搜索引擎优化）；另一个是不利于首屏渲染，增加了白屏时间

## Node.js 的角色
那么`Node.js`在上面那些过程中究竟扮演了什么样的角色，可以说从服务端到客户端，`Node.js`都有用武之地：

我们可以使用`Node.js`来**编写`API`，提供数据访问的接口**；也可以使用它来**写一个动态服务器，去实现服务端渲染**。除了本文中涉及的，`Node.js`可以做的远远不止于此，感兴趣的可以进一步看一下[2021年 Node.js 开发者调查报告](https://nodersurvey.github.io/reporters/)。


说明：文中的图全部来源于[Node.js, Express, MongoDB & More: The Complete Bootcamp 2022 | Udemy](https://www.udemy.com/course/nodejs-express-mongodb-bootcamp/)