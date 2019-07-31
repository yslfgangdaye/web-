koa 发布已经快 6 年的时间，作为继 express 之后 node 服务框架最大的黑马，有很多的设计思想值得我们学习，本文从简到繁逐步介绍 koa，同时适合新老手阅读。

## 介绍

这里引用中文官方网站的原文

> Koa 是一个新的 web 框架，由 Express 幕后的原班人马打造， 致力于成为 web 应用和 API 开发领域中的一个更小、更富有表现力、更健壮的基石。 通过利用 async 函数，Koa 帮你丢弃回调函数，并有力地增强错误处理。 Koa 并没有捆绑任何中间件， 而是提供了一套优雅的方法，帮助您快速而愉快地编写服务端应用程序。

既然是 web 框架大家一定不陌生，通过启动一个 node http server，监听一个端口，进而我们就可以通过类似 `localhost:3000` 在本地访问我们的服务了，这个服务可以是 web 网站，可以是 restful 接口，也可以是静态文件服务等等。

## Hello Word

任何语言、框架都存在 `Hello Word` 示例，来表达其最简单的入门 Demo，代码如下

![clipboard.png](https://segmentfault.com/img/bVbvoi7?w=472&h=338)

此时访问浏览器 `localhost:3000`，我们会看到打印出了 `Hello Word`，此时一个基于 koa 的服务就启动完成了。

## 上下文

理解 koa 第一步，搞清楚上下文的作用

例如：微信群里面有人说外面下雪了，你跑到窗边看到的却是晴空万里，这时你才意识到同样是 10 月份，他在寒冷的北方，你在酷暑的南方

类似的，一次请求会包含用户的登录状态，或者一些Token之类的信息，这些信息就是上下文的一部分，用于确定一次的请求环境

Koa 的 Context 把 node 的 request, response 对象封装进一个单独对象, 并提供许多开发 web 应用和 APIs 有用的方法. 那些在 HTTP server 开发中使用非常频繁操作, 直接在 Koa 里实现, 而不是放在更高层次的框架, 这样中间件就不需要重复实现这些通用的功能。

## 中间件

先来看一个官方的例子：

![clipboard.png](https://segmentfault.com/img/bVbvoi9?w=920&h=1062)

简单解释下，代码起始初始化一个 koa 实例，下面分别通过 use 方法载入了三个中间件方法，执行顺序：

1. 进入第一个中间件
2. `next()` 跳到下一个中间件
3. `new Data()` 记录当前时间
4. `next()` 跳到下一个中间件
5. 将 `ctx.body` 赋值
6. 回到上一个中间件再次记录当前时间并计算时间差存到 `http header` 中
7. 回到上一个中间件将 header 中的 `X-Response-time` 打印出来

这里的执行顺序延伸出了十分经典的洋葱模型

![clipboard.png](https://segmentfault.com/img/bVbvojb?w=478&h=435)

在一次请求的过程中会往返经过同一中间件两次，允许我们处理不同请求阶段的逻辑

## 源码解析

上面分别介绍了 koa 里面两个最重要的概念，下面我们分析下 koa 内部是如何运作的，所谓的洋葱模型是如何建立的

koa 源码的 lib 目录十分简单

```
lib
  |- application.js
  |- context.js
  |- request.js
  |- response.js
```

### Application 类初始化

入口文件是 `application.js`，我们先从这里入手

![clipboard.png](https://segmentfault.com/img/bVbvojg?w=970&h=630)

Application 是一个 class，这个类继承了 node 的 Events 这里不详细展开，在 `constructor` 中初始化了以下内容：

- proxy 代理默认不开启
- middleware 中间件是个空数组，这里重点注意下
- env 根据环境变量 `NODE_ENV` 来判断
- context、request、response 分别通过 `Object.create` 方法将 lib 目录下对应的文件导入到 `this` 当前上下文，且不污染引入对象

### use 方法

按照正常的编码顺序，在初始化完 koa 实例后（即 `const app = new Koa()`），我们需要调用 `app.use()` 去挂载我们的中间件，那么我们看下 use 方法做了什么

![clipboard.png](https://segmentfault.com/img/bVbvojk?w=1156&h=558)

判断中间件为 function，判断中间件是否为 generator function 类型，只是简单的将中间件函数 `push` 到了 `middleware`数组中。

此时心中有没有大写的 WHAT？

其实就是这么直白，没什么复杂逻辑，后面也许大家都猜到了，循环调用 `middleware` 中的方法去执行，此处尚未表明洋葱模型是怎么来的，我们先不展开，继续按代码顺序执行。

### listen 方法

按照正常的编码顺序，在 `use` 完我们的中间件之后就是 `app.listen(3000)`

一起看下这个 listen 干了什么

![clipboard.png](https://segmentfault.com/img/bVbvojm?w=988&h=342)

这里的 `http.createServer` 就是 node 原生启动 http 服务的方法，这里稍微扩展下基础知识，此方法接受两个参数

- `options[IncomingMessage, ServerResponse]` 这里从 node 版本 v9.6.0, v8.12.0 后才支持，这里不赘述
- `requestListener` 此参数为 function 类型，每次请求会传入 `req, res` 两个参数

不难理解这里的 `this.callback()` 方法一定是返回了一个函数，并且接收两个参数 `(req, res)`，下面看下源码

![clipboard.png](https://segmentfault.com/img/bVbvojC?w=1304&h=630)

这个 callback 中的信息量有点大，代码本身并不难理解，注释也有说明，**从这里展开从上到下分别解释**

### compose 方法

这里的 compose 方法主要负责生成洋葱模型，通过 `koa-compose` 包实现，源码如下

![clipboard.png](https://segmentfault.com/img/bVbvojG?w=1340&h=738)

从注释看得出大致逻辑，这里的巧妙之处在于 `fn(context, dispatch.bind(null, i + 1))`。

这个 `dispatch.bind(null, i + 1)` 就是我们通常写中间件的第二个参数 `next`。

我们执行这个 `next()` 方法实际上得到的是下一个中间件的执行。

也就不难理解为什么我们 `await next()` 的时候等待的是后面所有中间件串联执行后了，回头再看下上文中间件部分的执行顺序就豁然开朗了。

### createContext 方法

callback 中的展开解释，看下 `const ctx = this.createContext(req, res)` 做了什么

![clipboard.png](https://segmentfault.com/img/bVbvoj3?w=1224&h=666)

这里主要是将 req, res 及 this.request, this.response 都挂载到了 context 上，并通过赋值理清了循环引用层级关系，为使用者提供方便。

### handleRequest 方法

还是 callback 中的展开解释，看下 `this.handleRequest(ctx, fn)` 这部分做了什么

![clipboard.png](https://segmentfault.com/img/bVbvoj8?w=716&h=306)

分别拿到 ctx 和 compose 生成的洋葱模型，开始逐一消费中间件。

### context.js 文件

上面理清了整体框架，下面看下 `context.js` 内部的细节，在文件结尾有两大段的代理

![clipboard.png](https://segmentfault.com/img/bVbvokf?w=632&h=918)

![clipboard.png](https://segmentfault.com/img/bVbvokg?w=582&h=1242)

这里可以看到所有的 req 及 res 的方法集合，那么哪些方法可读，哪些可写，哪些既可读又可写，哪些方法不允许修改

这就是 `delegates` 这个库做的事情。

`delegates` 内部利用了，`__defineGetter__` 和 `__defineSetter__` 方法控制读写，当然我们可以从中学习思想，也不能盲从

这两个 api 去 MDN 上搜索会给出相同的警告信息

> This feature is deprecated in favor of defining setters using the object initializer syntax or the Object.defineProperty() API.

其实还是建议我们使用 vue 的代理方式 `Object.defineProperty()`，不过这个库有四年没更新了依然稳定运行着，还是深受 koa 开发者认可的。

### 其它

`request.js` 和 `response.js` 文件没什么可以讲，就是具体的工具方法实现，方便开发人员调用，感兴趣可以自行阅读源码。

## 应用

智联前端架构整体的 node 服务都基于 koa 实现，包括我们的 vue 服务端渲染和 node restful api 等等。

我们选择 koa 的原因是其本身轻巧，可扩展性良好，支持 async、await 的异步，彻底摆脱了回调地狱。

市面上也有成熟基于 koa2 的企业级解决方案，如 eggjs 和 thinkjs。

## 总结

揭开 koa 的神秘面纱，让开发者关注业务逻辑同时也关注下框架本身，有利于问题排查和编写扩展，与此同时可以学习 express、hapi 等同类型框架的思想，结合现有企业级解决方案，选一款适合你的框架，总之框架不论好坏，只论场景