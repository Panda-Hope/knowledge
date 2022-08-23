## 为什么选 Preact

> Fast 3kB alternative to React with the same ES6 API.
> React 的 3kb 轻量化方案，拥有同样的 ES6 API

- **体积小**。React v15.6.1 有 49.8kb，最新的 React v16.0 小了很多，有 34.8kb。而 Preact 官网声称只有 3kb，实测 v8.2.5 版本有
 4.1kb，但这已经比 React 小了至少 30kb，在移动端网页开发中占了不少优势。
- **性能高**。是最快的虚拟 DOM 框架之一，具体可以查看 [Results for js web frameworks benchmark – round 6](http://www.stefankrause.net/js-frameworks-benchmark6/webdriver-ts-results/table.html)。
- **生态好**。官方提供 [preact-compat](https://github.com/developit/preact-compat)，可以无缝使用 React 生态系统中的各类组件。
- **更方便**。相比 React，Preact 添加了几个更为便捷的特性，包括可以直接使用标准的 HTML 属性（如 `class` 和 `for`），`props`，`state` 和 `context` 作为参数传进了 `render()` 等。

除了 Preact，React-like 中比较出名的还有 [Inferno](https://infernojs.org/)，它大小在9kb左右。它和前面两者相比，还加了一个新的特性就是无状态组件（stateless components）也支持生命周期事件，从而可以减少 `class` 的使用，做到更加轻量。性能比 React 和 Preact 都要好，其它方面和 Preact 差不多。但是有个问题是 Inferno 用了一些相对较新的特性，比如 `Promise`、`Map`、`Set`、`WeakMap` 等，所以浏览器兼容性比 Preact 要差点。

## 迁移指南

### 官方文档

- [用 Preact 替换 React](https://preactjs.com/guide/switching-to-preact)

官方文档提供了2种途径，都是在原项目上把 React 替换成 Preact，我们为了保持项目干净采用创建新项目的方式来做。

### 准备

#### 1. 创建项目

Preact 官方提供了2种方式来创建项目：

1. [preact-cli](https://github.com/developit/preact-cli/)
2. [Preact Boilerplate / Starter Kit](https://github.com/developit/preact-boilerplate)

官方推荐用方式一，但我们为了方便定制采用了方式二来创建，去掉了 `preact-compat` 等暂时没用到库。

#### 2. 复制代码

创建完项目，把原项目中 `src` 目录下的代码（index.js、组件等）拷贝到新项目 `src` 目录下就可以了。

这样准备工作就做好了，当然现在项目还是跑不起来，会各种报错，接下来我们需要修改代码。

### 修改代码

#### react

把 `react` 替换成 `preact`：

```js
import React from 'react';
// =>
import { h } from 'preact';
```

Preact 的 `h()` 相当于 React 中的 `createElement()`，可以阅读 [WTF is JSX](https://jasonformat.com/wtf-is-jsx/) 了解更多。

```js
import { render } from 'react-dom';
// =>
import { render } from 'preact';
```

```js
import { Component } from 'react';
// =>
import { Component } from 'preact';
```

### redux

直接替换成 preact-redux 就可以，其它都一样：

```js
import { Provider } from 'react-redux';
// =>
import { Provider } from 'preact-redux';
```

### router

我们原项目采用的是 React Router v3，Preact 官方提供了 preact-router，它并不是 React Router 的 preact 兼容版本，而是另一种轻量级的路由方案。如果你还是想用 React Router，可以它的 v4 版本，因为 v4 版本可以直接和 Preact 一起使用，而不需要 preact-compat 来做兼容。

所以如果你有比较复杂的需求的话，比如路由嵌套、视图合成等等，可以考虑用 React Router v4；如果只是想要比较基本的路由需求的话，那就用 preact-router 好了。我们项目路由需求不复杂，为了追求轻量，就采用了 preact-router，毕竟 React Router 比 preact-router 大了很多。

因为不兼容，所以改动也是比较大的：

```js
import { Router, Route, IndexRoute } from 'react-router';
// =>
import Router from 'preact-router';
```

```js
import { hashHistory } from 'react-router';
// =>
import createHashHistory from 'history/createHashHistory';
const hashHistory = createHashHistory();
```

```js
const routes = (
  <Router history={hashHistory}>
    <IndexRoute component={Home} />
    <Route path="about" component={About} />
    <Route path="inbox" component={Inbox} />
  </Router>
);
// =>
const routes = (
  <Router history={hashHistory}>
    <Home path="/" />
    <About path="/about" />
    <Inbox path="/inbox" />
  </Router>
);
```

原 `<Route/>` 的 `onEnter`/`onLeave` 钩子也没了，不过没关系，因为用组件的生命周期才是更合理的。

这样做完路由配置之后，组件中用到路由的部分也不一样了。原先取当前页面的 URL 或者 query 是这样的：

```js
class Home extends Component {
  componentWillMount() {
    const { router } = this.context;
    const { params } = this.props;

    const url = router.location.pathname;
    const { id } = params;
  }
}
```

现在就只需要：

```js
class Home extends Component {
  componentWillMount() {
    const { url, id } = this.props;
  }
}
```

是不是更简单了。preact-router 引入了更少的 API，语法也更加简洁。

有了路由，当然也少不了链接。原先我们会写成这样：

```js
import { Link } from 'react-router';

<Link to="/about">关于</Link>
```

现在只需要写成普通的 `<a/>` 标签就可以了，因为 preact-router 会自动匹配 `<a/>` 标签到路由：

```html
<a href="/about">关于</a>
```

当然如果你要修改激活状态样式的话还是可以用 `<Link/>`，但注意这里用的是 `href="/"` 而不是 `to="/"`：

```js
import { Link } from 'preact-router/match';

<Link activeClassName="active" href="/">Home</Link>
```

## 总结

上面只是梳理了我们在做迁移时需要改动的地方，因为之前并没有用第三方的 React 组件，所以也没有用 preact-compat，除了 router 改得比之前更简单了，其它基本不需要怎么改动，整体来说迁移成本不高。

迁移后效果最明显的就是编译出来的 js 小了很多，基本是之前的1/3，所以移动端网页开发推荐用 Preact 替换 React。
