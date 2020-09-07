## 序言：
本篇是Node开发介绍的第一篇，主要是介绍Node开发框架 Koa以及Koa-router的运行原理。  
也是后续Egg.js(基于Koa衍生的Node企业级框架)篇章介绍的前站。

## 介绍：
#### Koa框架是什么?  
Koa是基于Node.js平台的下一代Web开发框架。  
Koa是一个轻量级的、更富有表现力的、可扩展性的高效便捷的Node开发框架。

他主要做了以下事情：
- 基于node原生req和res为request和response对象赋能，并基于它们封装成一个context对象
- 基于async/await（generator）的中间件洋葱模型机制

## 总览：
#### Koa是如何运行的？

<div align="space-between">
  <img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/koa.png" width="860" height="420">
</div>

以上便是Koa的运行机制图，主要分为以下几个步骤：
1. 构建Koa应用，并完成初始化
2. 基于HTTP请求的Req与Res，构建自身的Request与Response类，优化、增强HTTP请求的相关处理流程
3. 构建洋葱模型的中间件机制。例如Koa-router，来对HTTP请求进行分发，并处理返回相对应的结果

#### Koa包含那几个模块？
- Context(上下文运行环境)：  
	这是Koa运行的上下文环境，也是Koa Application类其自身，  
	主要包含了应用的初始化、服务器的启动、HTTP请求的处理、中间件的挂载这几个模块。
- Request(HTTP 请求封装)：  
	Request对象基于node原生req封装了一系列便利属性和方法，供处理请求时调用
- Response(HTTP 返回封装)：  
	Response对象与Request对象类似，除却一些基本属性的封装外，Response还提供了对数据返回、HTTP返回头设置、路由重定向等功能

#### 什么是Koa的中间件洋葱模型机制？
- 中间件洋葱图：
	<img src="https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67" alt="">
	- 中间件执行顺序图：
	<img src="https://raw.githubusercontent.com/koajs/koa/a7b6ed0529a58112bac4171e4729b8760a34ab8b/docs/middleware.gif">
	
	洋葱模型是指 通过ES6 Generator语法，所有的请求经过一个中间件的时候都会执行两次。  
	洋葱模型使得Koa在处理中间件后置逻辑上更加便捷、高效，也使得语法更为简洁明了。

## 模块介绍：
### Context(上下文运行环境)：
	这是Koa运行的上下文环境，也是Koa Application类其自身, 主要功能包含有：
	- 应用的创建
		Koa Constructor构造函数完成了Koa类的一些基本属性例如：proxy、subdomainOffset、maxIpsCount的初始化，  
		以及通过delegate来完成Context、Request以及Response类的预装载，以减轻后续每次HTTP请求所需创建的资源。
	- HTTP请求的监听与处理  
		在Koa，HTTP请求的处理由callBack以及handleRequest来处理：
	```$xslt
	/**
	   * Return a request handler callback
	   * for node's native http server.
	   *
	   * @return {Function}
	   * @api public
	   */
	
	  callback() {
	    const fn = compose(this.middleware);
	
	    if (!this.listenerCount('error')) this.on('error', this.onerror);
	
	    const handleRequest = (req, res) => {
	      const ctx = this.createContext(req, res);
	      return this.handleRequest(ctx, fn);
	    };
	
	    return handleRequest;
	  }
	
	  /**
	   * Handle request in callback.
	   *
	   * @api private
	   */
	
	  handleRequest(ctx, fnMiddleware) {
	    const res = ctx.res;
	    res.statusCode = 404;
	    const onerror = err => ctx.onerror(err);
	    const handleResponse = () => respond(ctx);
	    onFinished(res, onerror);
	    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
	  }
	```
	callback负责对API请求的处理，每当有请求接收到时，  
	callback将会基于Req和Res创建一个新的上下文作用域，并将中间件通过middleware进行组装，传递给handleRequest进行处理。  
	
	
	handleRequest被调用开始依序调用相应中间件，开始对API请求作出相应处理，  
	同时监听此过程中发生的错误，最后对HTTP返回的内容数据进行统一的格式化处理。  
	最终完成了一个API请求，到中间件处理，最后返回的过程。
	
- 中间件的装载  
	顾名思义，此模块是为了完成对于Koa中间件的装载而存在的，  
	在Koa中中间件是一个依序加载的队列，也因此装载的过程也十分简单，将需要执行的中间件推入队列中即可`this.middleware.push(fn)`。

### Request(HTTP 请求封装)：
Request类是Koa基于HTTP Req上进行的二次封装，主要包含以下功能：
- header 请求头的获取与设置
- 请求url、origin、href、method以及host等HTTP信息的访问
- IP、secure、subdomains、accept等辅助支持类相关信息的访问

### Response(HTTP 返回封装)：
Response类与Request类似，除却一些基本信息的获取，还增添了一些额外的功能：
- HTTP状态码以及Response Body的设置
- 路由的重定向redirect
- Content-Type、Last-Modified、ETag等HTTP响应头字段设置
- Socket流是否可写入检测等功能...

### 中间件(Koa-Router)：
在Koa中中间件便是核心所在，这是Koa实际上处理HTTP请求并返回相应结果的地方。  
这里我们将着重描述一下Koa-Router是如何运行的？

##### 首先我们看下Koa-Router的一些基本结构：
<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image.png" />
与Koa的中间件相同，Koa-Router的路由也同样是一个依次调用的队列，  
每当我们注册一个路由时，Koa-Router便会注册一个新的Route对象并将其推入Stack队列中，  
而每一层Route中会存在一个对应的Layer，Layer负责对路由规则进行解析与匹配(遵循path-to-regexp规则)，并执行对应路由上的函数操作。


在了解到路由的一些基本结构后，我们来看看Koa-Router还有哪些其他特点：
- 路由前缀补齐：
	```javascript
  router.prefix('/things/:thing_id')
  ```
- 路由嵌套：
	```javascript
	var forums = new Router();
	var posts = new Router();
	
	posts.get('/', (ctx, next) => {...});
	posts.get('/:pid', (ctx, next) => {...});
	forums.use('/forums/:fid/posts', posts.routes(), posts.allowedMethods());
	// responds to "/forums/123/posts" and "/forums/123/posts/123"
	app.use(forums.routes());
	```
- Options请求与405、501响应处理：
	```javascript
	var Koa = require('koa');
	var Router = require('koa-router');
	
	var app = new Koa();
	var router = new Router();
	
	app.use(router.routes());
	app.use(router.allowedMethods());
	```
- Param路由参数生成指定路由：
	```javascript
	router
	.param('user', (id, ctx, next) => {
	  ctx.user = users[id];
	  if (!ctx.user) return ctx.status = 404;
	  return next();
	})
	.get('/users/:user', ctx => {
	  ctx.body = ctx.user;
	})
	.get('/users/:user/friends', ctx => {
	  return ctx.user.getFriends().then(function(friends) {
	    ctx.body = friends;
	  });
	})
	// /users/3 => {"id": 3, "name": "Alex"}
	// /users/3/friends => [{"id": 4, "name": "TJ"}]
	```
	

## 总结：
Koa是个高效、轻量级的Node开发框架，Koa的源码并不复杂，却刚好能够满足HTTP应用开发中的基本功能。  
基于async/await（generator）的中间件洋葱模型使得Koa不仅能够摆脱传统Node开发中回调地狱的问题，  
也使得在此之上的扩展更为便捷，这也为基于Koa开发的Egg.js框架提供了足够的基本功能。
