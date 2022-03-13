## 前言
前面第二章，我们讲述了 __Reactive__ 响应式数据是如何去实现双向绑定的，但是却没有去叙述`Effect`最终是如何去完成视图更新的，  
这里我们将结合 __Vue.js__ 的生命周期，从应用构建开始去更加完整的讲述一个 __Vue.js__ 应用是如何完成构建并运行的。

## Vue.js应用的生命周期
首先，我们来看下 __Vue__ 官方所给出的生命周期示意图，这里我们将整个生命周期简单的分为四个阶段：
1. 应用构建阶段
2. 模板编译阶段
3. 运行阶段
4. 卸载阶段

本篇文章，我们将去讲述 __Vue.js__ 在应用构建阶段与运行阶段的一些操作与细节，以期为读者对 __Vue__ 的运行机制带来一个更加深入的了解。

<image src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/lifecycle.16e4c08e.png" align="center" width="80%" />

## 一步一步构建Vue.js应用

### createApp创建应用
在 __Vue.js 3.0__ 中 __Vue__ 使用`createApp`函数来替代了原有的构造函数来创建应用。那么让我们首先来看下 __Vue__ 在这里做了那些事情：

#### 构建应用上下文环境
```typescript
const context = createAppContext()

function createAppContext(): AppContext {
  return {
    app: null as any,
    config: {
      isNativeTag: NO,
      performance: false,
      globalProperties: {},
      optionMergeStrategies: {},
      errorHandler: undefined,
      warnHandler: undefined,
      compilerOptions: {}
    },
    mixins: [],
    components: {},
    directives: {},
    provides: Object.create(null),
    optionsCache: new WeakMap(),
    propsCache: new WeakMap(),
    emitsCache: new WeakMap()
  }
 }
```
在这里，我们得到了一个 __Vue.js__ 应用的基本骨架，它包含基本的应用、配置、component等属性。

#### 创建应用主体
```typescript
const app: App = {
  _uid: uid++,
  _component: rootComponent as ConcreteComponent,
  _props: rootProps,
  _container: null,
  _context: context,
  _instance: null,
  version,
  use: () => App,
  mount: (
    rootContainer: HostElement,
    isHydrate?: boolean,
    isSVG?: boolean
  ) => any,
  unmount: () => void,
  provide: (key, value) => App
}
```
`createApp`函数的第二步，主要是导出了`mount`函数，在这里 __Vue__ 首先为组件构造了 `vnode`，这是整个`Virtual DOM`构建的的第一步：“创建`vnode`节点”，之后我们开始执行`render函数`正式开始我们的应用构建。

#### BeforeCreate与Created生命周期

__Vue__ 将原有的 __Vue 2.0__ 的构造函数移动到`applyOptions`函数中，以兼容原有的功能 __API__ ，在这里 __Vue__ 首先会调用 `beforeCreate` 钩子，  

在完成数据响应式处理、函数包装、计算属性以及观察器等参数初始化之后， __Vue__ 完成了应用组件的创建，开始调用`created`钩子函数，来完成整个应用的初始化操作。

```typescript
export function applyOptions(instance: ComponentInternalInstance) {
  // 当options参数被处理之前调用 beforeCreate 钩子
  callHook(options.beforeCreate, instance, LifecycleHooks.BEFORE_CREATE)
  
  // 处理函数封装
  methodHandler.bind(publicThis)
  
  // 响应式数据处理
  instance.data = reactive(data)
  
  // 计算属性转换
  const c = computed({
    get,
    set
  })
  
  // 创建观察器
  createWatcher(watchOptions[key], ctx, publicThis, key)
  
  // 初始化完成，调用created生命周期钩子 
  callHook(created, instance, LifecycleHooks.CREATED)
}
```


### patch与compile

在完成了一个 __Vue.js__ 组件应用的构建之后，我们开始进入了编译环节，这里我们同样也需要去介绍 __Vue__ 源码中的两个模块 `pathch`与`compile`，  
让我们先来看下`patch`是什么？

#### 什么是patch

当 __Vue.js__ 完成了应用的构建之后，我们得到了组件应用的`virtual dom`，众所周知`virtual dom`在`MVVM`框架中是十分重要的一环，  




## Vue.js是如何运行的？

## 总结
下一篇，我们将开始介绍 __Vue__ 的 `Compile`编译器，来讲述 __Vue__ 是如何去完成模板编译的。

## 文献参考
[Vue3 source code analysis (5): Patch algorithm](https://segmentfault.com/a/1190000040097158/en)  
[Vue3 framework principle realization (three)-patch](https://www.mo4tech.com/vue3-framework-principle-realization-three-patch.html)
