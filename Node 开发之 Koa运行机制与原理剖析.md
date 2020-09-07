## 序言：
本篇是Node开发介绍的第一篇，主要是介绍Node开发框架 Koa以及Koa-router的运行原理。也是后续Egg.js(基于Koa衍生的Node企业级框架)篇章介绍的前站。

## 介绍：
##### Koa框架是什么?  
Koa是基于Node.js平台的下一代Web开发框架。  
Koa是一个轻量级的、更富有表现力的、可扩展性的高效便捷的Node开发框架。

他主要做了以下事情：
- 基于node原生req和res为request和response对象赋能，并基于它们封装成一个context对象
- 基于async/await（generator）的中间件洋葱模型机制

## 总览：
##### Koa是如何运行的？

<div align="space-between">
  <img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/koa.png" width="860" height="420">
</div>

以上便是Koa的运行机制图，主要分为以下几个步骤：
1. 构建Koa应用，并完成初始化
2. 基于HTTP请求的Req与Res，构建自身的Request与Response类，优化、增强HTTP请求的相关处理流程
3. 构建洋葱模型的中间件机制。例如Koa-router，来对HTTP请求进行分发，并处理返回相对应的结果

##### Koa包含那几个模块？
- Context(上下文运行环境)：  
	这是Koa运行的上下文环境，也是Koa Application类其自身，主要包含了应用的初始化、服务器的启动、HTTP请求的处理、中间件的挂载这几个模块。
- Request(HTTP 请求封装)：  
	Request对象基于node原生req封装了一系列便利属性和方法，供处理请求时调用
- Response(HTTP 返回封装)：  
	Response对象与Request对象类似，除却一下基本属性的封装外，Response还提供了对数据返回、HTTP返回头设置、路由重定向等功能

##### 什么是Koa的中间件洋葱模型机制？
- 中间件洋葱图：
<img src="https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67" alt="">
- 中间件执行顺序图：
<img src="https://raw.githubusercontent.com/koajs/koa/a7b6ed0529a58112bac4171e4729b8760a34ab8b/docs/middleware.gif">

通过ES6 Generator语法 所有的请求经过一个中间件的时候都会执行两次，洋葱模型使得Koa在处理中间件后置逻辑上更加便捷、高效

## 模块介绍：
##### Context(上下文运行环境)：

##### Request(HTTP 请求封装)：

##### Response(HTTP 返回封装)：

##### 中间件(Koa-Router)：
