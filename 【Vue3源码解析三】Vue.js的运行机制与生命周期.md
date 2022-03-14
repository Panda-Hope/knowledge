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

## createApp创建应用
在 __Vue.js 3.0__ 中 __Vue__ 使用`createApp`函数来替代了原有的构造函数来创建应用。那么让我们首先来看下 __Vue__ 在这里做了那些事情：

### 构建应用上下文环境
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

### 创建应用主体
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

### BeforeCreate与Created生命周期

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


## patch与virtual dom构建

### 什么是patch?

当 __Vue.js__ 完成了应用的构建之后，我们得到了组件应用的`virtual dom`，众所周知`virtual dom`在`MVVM`框架中是十分重要的一环，  

`virtual dom`是应用组件向实际`HTML`节点渲染的中间体，也是`MVVM`框架对于整个模板渲染性能优化的核心，  

而其中`patch`则是负责`virtual dom`的基础节点`Vnode`的创建与更新,`patch`具有如下功能：

1. 创建或删除`Vnode`节点
2. 更新`Vnode`节点

### patch源码一览

```typescript
const patch: PatchFn = (
  n1,
  n2,
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  isSVG = false,
  slotScopeIds = null,
  optimized = false
) => {
  // patching & 不是相同类型的 VNode，则从节点树中卸载
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1)
    unmount(n1, parentComponent, parentSuspense, true)
    n1 = null
  }
    // PatchFlag 是 BAIL 类型，则跳出优化模式
  if (n2.patchFlag === PatchFlags.BAIL) {
    optimized = false
    n2.dynamicChildren = null
  }

  const { type, ref, shapeFlag } = n2
  switch (type) { // 根据 Vnode 类型判断
    case Text: // 文本类型
      processText(n1, n2, container, anchor)
      break
    case Comment: // 注释类型
      processCommentNode(n1, n2, container, anchor)
      break
    case Static: // 静态节点类型
      if (n1 == null) {
        mountStaticNode(n2, container, anchor, isSVG)
      }
      break
    case Fragment: // Fragment 类型
      processFragment(/* 忽略参数 */)
      break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) { // 元素类型
        processElement(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      } else if (shapeFlag & ShapeFlags.COMPONENT) { // 组件类型
        processComponent(/* 忽略参数 */)
      } else if (shapeFlag & ShapeFlags.TELEPORT) { // TELEPORT 类型
        ;(type as typeof TeleportImpl).process(/* 忽略参数 */)
      }
  }
}
```

在这里`patch`接收了两个`node`节点，n1是旧节点、n2是新节点，当新旧节点不同时，旧的节点将会被直接卸载，  

当处于`BAIL`模式时，性能优化将会被关闭，因为此时是初次进入，接下来`patch`将会依照不同的类型执行对应的操作。

当处理元素节点时，`patch`会去执行两步操作：
1. 比较元素节点
2. 更新节点数据数据

对于元素的更新将会分为以下几种具体情况：
```typescript
switch(patchFlag) {
  case FULL_PROPS: // 表示元素需要全面更新，比如dynamic key
  case CLASS: // 当属于class类型时,patch会进行比较判断并进行进行
  case STYLE: // 样式类型，样式将会会自动添加到节点
  case PROPS: // props传参将会被提取成数组，并执行遍历将新旧数据进行对比，然后更新参数
  case TEXT: // 文本类型将会被直接替换
}
```
当执行完`patch`后 __Vue__ 将会调用`beforeMount`钩子，并开始进行模板编译。

### diff算法

__Vue__ 的`diff`算法是用来决定`vnode`节点是否需要更新的判断逻辑，这里我们简单阐述一下`diff`算法的实现逻辑。  

首先 __Vue__ 会执行两个前置操作以便于后续算法执行：

1. 将`vnode`节点打上`key`标签将节点分为带`key`标签的与不带`key`标签的两种
2. 将`vnode`节点树折平成一维数据，因为队列的遍历在执行效率上优于树形的遍历

而具体的算法执行则主要分为带`key`值与不带`key`值的两种：

#### 无key值情况

此种情况下的判断最为简单，因为再次之前 __Vue__ 已经将节点树进行树遍历，并依序排列完成，  

因此此时仅需要取得新旧两个节点树的公共长度`commonLength = Math.min(old.length, new.length)`然后将公共节点之外的节点进行删除或者新增即可。

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/e85b794f1950017098081825790d9b74.jpg" />

#### 有key值情况

此时情况相比之前的便要复杂得多，这里主要分为四种不同的情形：

1. 前序相同
2. 后序相同
3. 中序相同
4. 乱序排序

对于前面三种类型而言，其本质与之前的无`key`值排序相差无多，最为核心的是在于乱序排序这种情况，  

__Vue__ 首先会尽可能的将节点不断的重复前面三种判断，最大限度的找寻出其最大公约数，在此之后我们所剩下的便是最后的乱序节点处理了。

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/7ac7eaf09307e8338e9e9af5efa43f7c.jpg" />

而对于乱序节点而言 __Vue__ 采用了以下不同策略的方式来实现其更新：

1. 构建`key:index`映射关系，比如常见的`v-for`指令中`key`，利用`key`值的变化判断其是否需要更新
2. 遍历数组，更新新旧`vnode`数组中相同的节点，同时卸载不再使用的废弃节点
3. 新旧节点数组中存在可重复使用的交叉节点，将重复的交叉节点进行位移操作，减少操作次数。

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/back.b7c62a1.png" />



## compile模板编译




## Vue.js是如何运行的？

## 总结
下一篇，我们将开始介绍 __Vue__ 的 `Compile`编译器，来讲述 __Vue__ 是如何去完成模板编译的。

## 文献参考
[Vue3 source code analysis (5): Patch algorithm](https://segmentfault.com/a/1190000040097158/en)  
[Vue3 framework principle realization (three)-patch](https://www.mo4tech.com/vue3-framework-principle-realization-three-patch.html)  
[graphical-analysis-of-vue3-diff-algorithm.html](https://programmer.ink/think/graphical-analysis-of-vue3-diff-algorithm.html)
