## 前言
前面第二章，我们讲述了 __Reactive__ 响应式数据是如何去实现双向绑定的，但是却没有去叙述`Effect`最终是如何去完成视图更新的， 

这里我们将结合 __Vue.js__ 的生命周期，从应用构建开始去更加完整的讲述一个 __Vue.js__ 应用是如何完成构建并运行的。  

本章我们较为详细的介绍了 __Vue.js__ 在运行中所遇到的一些核心模块功能，全文较为冗长，还请慢品o(╯□╰)o。


## Vue.js应用的生命周期
首先，我们来看下 __Vue__ 官方所给出的生命周期示意图，这里我们将整个生命周期简单的分为四个阶段：
1. 应用构建阶段
2. 模板编译阶段
3. 运行阶段
4. 卸载阶段

本篇文章，我们将去讲述 __Vue.js__ 在应用构建阶段与运行阶段的一些操作与细节，以期为读者对 __Vue__ 的运行机制带来一个更加深入的了解。

<image src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/lifecycle.16e4c08e.png" align="center" width="80%" />

## createApp创建应用
在 __Vue.js 3.0__ 中 __Vue__ 使用`createApp`函数来替代了原有的构造函数来创建应用，那么让我们首先来看下 __Vue__ 在这里做了那些事情：

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
`createApp`函数的第二步，主要是导出了`mount`函数，在这里 __Vue__ 首先为组件构造了 `vnode`，这是整个`Virtual DOM`构建的的第一步：  

“创建`vnode`节点”，之后我们开始执行`render函数`正式开始我们的应用构建。

### BeforeCreate与Created生命周期

__Vue__ 将原有的 __Vue 2.0__ 的构造函数移动到`applyOptions`函数中，以兼容原有的功能 __API__ ，在这里 __Vue__ 首先会调用 `beforeCreate` 钩子，  

在完成数据响应式处理、函数包装、计算属性以及观察器等参数初始化之后， __Vue__ 完成了应用组件的创建，  

开始调用`created`钩子函数完成整个应用的初始化操作。

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

因此此时仅需要取得新旧两个节点树的公共长度`commonLength = Math.min(old.length, new.length)`，  

然后将公共节点之外的节点进行删除或者新增即可。

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

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/bee8583bece95b43364b9730a7e3543b.jpg" />


## compile模板编译

在正式开始介绍`compile`编译器之前，我们先介绍一下 __Vue__ 的`beforemount`与`mounted`生命周期钩子，之后我们开始阐述`compile`编译器概念及其基本机制。

### BeforeMount与Mounted生命周期 

对于模板的编译 __Vue__ 实际上在执行`beforemount`生命周期钩子之前就已经完成了，  

对于`beforemount`与`mounted`钩子而言他们的区别仅仅在于将`node`节点挂载到`container`上面而言。

```typescript
// 实际开始编译模板处
Component.render = compile(template, finalCompilerOptions)

// beforeMount生命周期
invokeDirectiveHook(vnode, null, parentComponent, 'beforeMount')

// 挂载节点
hostInsert(el, container, anchor) 

// mounted生命周期
invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
```

### 什么是compile编译器？

编译器是一个非常宽泛的概念，从广义上来讲编译器会将某种编程语言写成的源代码（原始语言）转换成另一种编程语言，其主要目的是将人类能够阅读的语法翻译成机器能够阅读的语言。  

在编译原理一书中完整的编译器包括语法、词法定义以及自动机、语法制导翻译、词法解析、语法分析树等模块，而从过程角度来说则分为：  

`文法定义->词法分析->语法分析->语法分析树构建->语法制导翻译->代码优化->最终代码生成`，这几个步骤。  

无论何种编译器都离不开语法分析，中间件构建，代码生成这三个环节， __Vue.js__ 的编译器亦是如此。

### 语法分析

`compile`的语法分析由`baseParse`函数完成，语法分析主要分为两步：
1. 解析模板构建AST语法抽象树
2. 处理style、class、diretive等辅助功能

```typescript
function baseParse(
  content: string,
  options: ParserOptions = {}
): RootNode {
  const context = createParserContext(content, options)
  const start = getCursor(context)
  return createRoot(
    parseChildren(context, TextModes.DATA, []),
    getSelection(context, start)
  )
}
```

### 中间件-AST语法树
`AST语法分析树`具体三种阶段：
1. 初始阶段，此时的`AST语法分析树`由前置的语法分析构建而成，此时它仅由基本的`vnode`节点组成，比较粗糙。
2. 之后`AST语法分析树`将会经过优化标记为带`key`与不带`key`的节点，以便于我们前面提到过的`diff`算法的实现
3. 最终阶段，此时的`AST语法分析树`再次经过处理，生成可执行代码，例如 _c、_l 之类的。

### 代码生成

最后`compile`将会整个编译所得到的结果打包成一个`render`函数，以便于后续`mount`操作的执行。

```typescript
export type RenderFunction = () => VNodeChild
```

## Vue.js是如何运行的？

在经过了应用的创建与挂载之后，我们得到了完整的可运行的 __Vue.js__ 应用，此时我们便到了本章的最后一个模块，应用是如何运行的？

如果说将`Scheduler`比作一个车站的调度室，那么`Effect`则是负责拖运货物的列车，而车上拖运的货物之一便是组件的刷新函数`render`。

### 组件是如何去更新的？

我们在前面的 __Reactivity__ 模块时便已经提过，`Dep`是整个 __Vue.js__ 工厂的`搬运工`，它们负责将货物装载到`Effect`这趟列车，  

然后由`EffectScope`这个列车长通知调度室`Scheduler`来发车，将货物运送到工厂由`SetupRenderEffectFn`来真正执行组件的更新，这个步骤共分为：

1. `Dep`获取依赖的更新，通知`Effect`数据需要更新
2. `Effect`收集依赖的更新数据情况将之汇总
3. 由`EffectScope`这个总列车头来判断哪些`Effect`需要装载更新
4. 调用`Scheduler`将本次组件更新的全部操作推入下一个微任务队列中
5. 执行`setupRenderEffect`函数完成组件更新，在下一个微任务队列中调用`updated`钩子，通知组件更新完成

<img width="500" align="center" src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/%E6%88%AA%E5%B1%8F2022-03-16%20%E4%B8%8B%E5%8D%882.44.47.png" />

### SetupRenderEffectFn组件更新函数

`SetupRenderEffectFn`函数是实际负责组件更新的地方，在这里 __Vue__ 会将新旧节点树进行`diff`对比生成新的节点树，  

并为此次更新创建一个新的`effect`，在绑定作用域之后调用`effect.run`开始执行更新操作，最后在下一个微任务队列中执行`updated`钩子。

```typescript
const setupRenderEffect: SetupRenderEffectFn = (
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG,
    optimized
  ) => {
  const componentUpdateFn = () => {
    let { next, bu, u, parent, vnode } = instance
    let originNext = next
    let vnodeHook: VNodeHook | null | undefined
    
    toggleRecurse(instance, false)
    
    if (next) {
      next.el = vnode.el
      updateComponentPreRender(instance, next, optimized)
    } else {
      next = vnode
    }

    // 调用before update钩子
    if (bu) {
      invokeArrayFns(bu)
    }
    
    // 将新旧 AST语法树进行对比更新
    patch(
      prevTree,
      nextTree,
      // parent may have changed if it's in a teleport
      hostParentNode(prevTree.el!)!,
      // anchor may have changed if it's in a fragment
      getNextHostNode(prevTree),
      instance,
      parentSuspense,
      isSVG
    )
    
    next.el = nextTree.el
    
    if (originNext === null) {
      // self-triggered update. In case of HOC, update parent component
      // vnode el. HOC is indicated by parent instance's subTree pointing
      // to child component's vnode
      updateHOCHostEl(instance, nextTree.el)
    }
    
    // 在下一个微任务队列中执行updated钩子
    if (u) {
      queuePostRenderEffect(u, parentSuspense)
    }
    // onVnodeUpdated
    if ((vnodeHook = next.props && next.props.onVnodeUpdated)) {
      queuePostRenderEffect(
        () => invokeVNodeHook(vnodeHook!, parent, next!, vnode),
        parentSuspense
      )
    }
  }
  
  // 创建render effect
  const effect = (instance.effect = new ReactiveEffect(
    componentUpdateFn,
    () => queueJob(instance.update),
    instance.scope // track it in component's effect scope
  ))
  
  const update = (instance.update = effect.run.bind(effect) as SchedulerJob)
  update.id = instance.uid
  
  // 开始执行更新
  update()
}
```

### BeforeUpdate与Updated生命周期

`BeforeUpdate`钩子调用于组件刷新之前，值得一提的是`Updated`钩子函数并非直接调用与`patch`更新之后，而是执行在下一个微任务队列，  

这是因为`patch`函数执行完成后，实际DOM的更新是在下一个宏任务中执行，因此需要在下一个微任务队列中执行 `Updated`钩子才能确保此时组件已完全实现更新。

```typescript
// beforeUpdate钩子
instance.emit('hook:beforeUpdate')

// updated钩子
queuePostRenderEffect(
  () => instance.emit('hook:updated'),
  parentSuspense
)
```

## 总结

本章，我们较为详细的介绍了一个完整的 __Vue.js__ 应用从应用主体创建、AST虚拟DOM搭建、模板编译最后到应用运行的过程，  

并介绍了各个模块中的核心部分`patch`、`diff算法`等，同时我们也提到了一些同样重要的辅助模块如：`dep`、`effect`、`scheduler`等，  

以期让读者对于整个 __Vue.js__ 的运行有着更加深刻的体会。本章比较冗长同时也是整个 __Vue3 源码解析系列__ 中最为重要的环节，

接下来我们将开始详细介绍 __Vue__ 的一些核心的辅助模块，之后我们会开始详细的讲解 __Vue__ 的编译器是如何去实现的。

## 文献参考
[Vue3 source code analysis (5): Patch algorithm](https://segmentfault.com/a/1190000040097158/en)  

[Vue3 framework principle realization (three)-patch](https://www.mo4tech.com/vue3-framework-principle-realization-three-patch.html)  

[graphical-analysis-of-vue3-diff-algorithm.html](https://programmer.ink/think/graphical-analysis-of-vue3-diff-algorithm.html)  

[Compiler principle and optimization strategy in vue3](https://copyfuture.com/blogs-details/202201291044027584)  

[Analysis of Vue compiling principle](https://qdmana.com/2022/02/202202020428204975.html)  

[编译原理第二版 [Compilers:Principle,Techniques and Tools]](https://item.jd.com/10058776.html)
