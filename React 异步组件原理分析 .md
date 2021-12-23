在分析可视化工具制作的时候发现组件库的组件是按需加载的，只有在实际使用到该组件的时候，才会主动加载该组件，对于这个功能中使用到的异步加载组件的是如何实现的呢？你能想到业务开发中哪些地方还用到了异步组件加载吗？

本文主要围绕以下内容从使用到源码分析：
- umi dynamic -> react-loadable -> import
- React.lazy and React.suspense

## Umi's dynamic
umi 基本 API 中提供了一个异步加载组件的方法 `dynamic` 。


### 使用案例
在工作中开发了一个业务组件 WhiteList 需要在使用的时候才加载而不是项目初始化的时候就加载，业务代码简化如下。

```javascript
// dynamic 组件

import { dynamic } from 'umi';

export default ({ component }) => dynamic({
  loader: async () => {
    // 这里的注释 webpackChunkName 可以指导 webpack 将该组件 WhiteList 以这个名字单独拆出去
    const { default: component } = await import(/* webpackChunkName: "external_A" */ `./components/${component}`);
    return component;
  },
})
```

```javascript
// dynamic 组件的具体使用

import Dynamic from './dynamic';

export default () => <Dynamic component="WhiteList" />
```

封装好一个 Dynamic 基本组件，在使用的时候我们直接传入需要异步加载的组件名就可以使用了，为什么只能异步加载相对目录 components 下的组件呢？

### 源码分析

[dynamic.tsx](https://github.com/umijs/umi/blob/master/packages/runtime/src/dynamic) 中可以分析出使用了 loadable 是在 `react-loadable` 的基础上修改而来，loadable 的作用是什么又是如何实现的呢？

```javascript
Loadable({
  loader: () => import(/* component module path */),
  loading: () => <div>loading</div>
})
```

loadable 的 loader 支持 function 和 map ，每个 loader 中都是用了动态导入 [`import`](https://github.com/tc39/proposal-dynamic-import)，实际异步的实现已经是提议中的一个功能了，react-loadable 只是处理了异步过程中的一些异常问题以及一些功能优化。

在 react-loadable 中通过状态判断，默认组件加载完成前会展示 loading 组件，因为组件的加载是返回的一个 promise ，通过成功以及异常来判断组件加载的后续展示。

#### import

在这之前我们可以通过 script 来加载模块，大致写法如下：

```javascript
function importModule(url) {
  return new Promise((reolve, reject) => {
    const script = document.createElement('script');
    const tempGlobal = `_tempModule${Math.random().toString(32).substring(2)}`;
    script.type = 'module';
    script.textContent = `import * as m from '${url}'; window.${tempGlobal} = m;`;

    script.onload = () => {
      resolve(window[tempGlobal]);
      delete window[tempGloal];
      script.remove();
    }

    script.onerror = () => {
      reject(window[tempGlobal]);
      delete window[tempGlobal];
      script.remove();
    }

    document.documentElement.appendChild(script);
  })
}

```
上诉写法能够在浏览器端加载 URL 形式的模块，不能通过模块名直接加载，而且 script 是在浏览器环境下使用，Node 环境中就不能正常使用了，import 的提议可以规范化来解决这个问题从而实现模块加载。其中 module path 必须是一个字符串或者模板字符串，不能是变量。

```javascript
import(/* module path or name */)
```

#### loader

import 返回的是一个 promise


#### code-splitting

项目使用了 webpack 打包工具，在 webpack 2+以后的版本都会通过 import() 自动代码分割代码，这样就不需要关心代码如何分割，只是在业务模块与组件拆分的时候考虑清楚模块划分。

```javascript
// main.js
import('./a')

// a.js
console.log('a.js');
```

通过 webpack 打包以后会自动生成分割文件 main.bundle.js 和 [chunkId].main.bundle.js，在 main.bundle.js 中通过 document.createElement('script') 来实现了加载 [chunkId] 对应的子文件。实际就是对前面的 mock import 的实际应用。

### React.lazy

lazy 是官方提供的一个允许动态导入组件的函数。我们可以不使用 Umi'Dynamic ，也可以不再使用 React Loadable 。直接使用官方提供的 lazy 函数就可以了，但是还是需要借助打包工具来实现 code splitting。

#### 项目应用

```javascript
import React, { lazy, Suspense } from 'react';
import { Loading } from '@/components'; // import loading component

const LazyComponent = lazy(() => import(/* component path */));

export default () => (
  <Suspense fallback={Loading}>
    <LazyComponent />
  </Suspense>
)
```
这里 lazy 需要配合 Suspense 使用，同样的我们可以包装一个 Dynamic。

```javascript
// Dynamic.jsx
import React, { lazy, Suspense } from 'react';
import { Loading } from '@/components';

export default loader => {
  const DynamicComponent = lazy(loader);
  return (
    <Suspense fallback={Loading}>
      <DynamicComponent />
    </Suspense>
  )
}

// demo.js 使用
import { Dynamic } from '@/utils';

export default () => Dynamic(() => import('./components/LazyComp'));
```

#### 源码解析

[react 17.0.2](https://github.com/facebook/react/blob/f15f8f64bbc3e02911d1a112fa9bb8f7066a56ee/packages/react/src/ReactLazy.js#L98) 中我们看到 lazy 方法接收了一个参数 ctor  。返回的是一个虚拟 DOM 对象。

```javascript
// react/src/ReactLazy 简化后
function lazy(cotr) {
  return {
    $$typeof: REACT_LAZY_TYPE,
    _payload: {
      _status: -1,
      _result: ctor,
    },
    _init(payload) {
      // ...
    },
  }
}
```

在 beginWork 中判断了 workInProgress.tag 如果类型是 LazyComponent ，执行 mountLazyComponent 函数，因为是异步函数 _current 一开始是 null ，会先初始化 lazyComponent。到这里就和上面的 lazy 函数的返回值关联上了。

```javascript
// react-reconciler/src/ReactFiberBeginWork 简化后
function beginWork(/* ... 参数省略 */) {
  // ... 省略前面部分代码
  switch(workInProgress.tag) {
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        updateLanes,
        renderLanes,
      );
    }
  }
  // ... 省略后面部分代码
}

function mountLazyComponent(/* ... 参数省略 */) {
  if (_current !== null) {
    // 重置状态
    // ...省略部分代码
  }

  // ... 省略部分前面代码

  // 这里使用的参数来源于 lazy 函数的返回值
  const payload = lazyComponent._payload;
  const init = lazyComponent._init;
  let Component = init(payload);
  // ... 省略部后面代码
}
```

因为这里主要是分析异步组件的使用与加载到最后的渲染，过程中相关的部分，暂时不扩展其他细节部分源码。组件加载过程中我们可以使用一个过渡效果，也就是 Suspense 组件通过高阶组件的形式判断如果当前异步组件的状态是 pending 状态时先渲染 fallback 组件，resolve 后再渲染异步组件。未来 Suspense 组件还会支持更多异步数据场景不仅仅是异步组件。

同 lazy 一样 Suspense 是在 beginWork 中判断类型 SuspenseComponent 然后执行 updateSuspenseComponent 函数，内部调用 mountChildFibers 挂载 然后 reconcileChildFibers。

```javascript
// - beginWork
//    - updateSuspenseComponent
//        - mountChildFibers Symbol(react.suspense)
//        - reconcileChildFibers Symbol(react.suspense)
```

#### ssr

lazy 和 Suspense 暂时不支持 ReactDOMServer。如果在 ssr 中使用可以参考使用 [Loadable Components](https://github.com/gregberge/loadable-components)。


### 文末思考

通过本次学习我们认识了什么是异步组件，从 umi's dynamic 和 官方提供的 lazy 和 Suspense 的使用到源码分析，大致知道了异步组件的加载过程的状态切换，再到 es 规范中目前处于提议中的 import 动态加载函数的实现，以及了解了 webpack 通过 import 来实现的自动分割代码。到这里也就可以想到文章开始的一个扩展提问了，在工作中我们通常也会在路由加载的时候使用异步组件来按需加载做项目优化。  
通过一个项目中的实际问题我们扩展学习了一个知识点，为今后的工作也提供了更好的理论支持。