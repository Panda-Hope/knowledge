## Reactive模块的作用
本章我们将讲述在 __Vue.js__ 中 __Reactive__ 响应式数据是如何去构建的，与 __Vue.js 2.0__ 相比 __Vue.js 3.0__ 在底层仍然使用通过 __Dep__ 进行数据收集依赖更新的方式来完成响应式数据处理。 但与之不同的是 __Vue.js 3.0__ 新增了 __Ref__ 数据结构，并使用了 __ReactiveEffect__ 来替换以往使用 __Watcher__ 来监听并更新数据的方式，同时增加了新的 __effectScope__ 作用域来提升性能。接下来让我们详细介绍 __Vue.js 3.0__ 所带来的新的响应式数据处理变化。

## Reactive与Ref
### Reactive
与 __Vue.js 2.0__ 返回一个创建 __Data__ 响应式数据的函数相比，__Vue.js 3.0__ 则是采用了使用 __reactive()__ 函数的方式来创建一个新的响应式数据对象，如下：
```typescript
function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
```
通过 __reactive__ 函数所创建的数据都是`unwrapperd`，同时 __reactive__ 所创建返回的对象将会使用 __Proxy__ 来构造一个新的对象来避免依赖更新的问题，这也就意味着：
```typescript
const obj = {}
reactive(obj) != obj // 返回新的Proxy对象
```
与 __Vue.js 2.0__ 相同，__Reactive__ 中的数据同样都是`deep`深度转换的。  

### Ref
在 __Vue.js 3.0__ 中，增加了一种新的数据结构类型 __Ref__，使用 __Ref__ 所构造的数据都是`wrapped`，这意味着你可以分配一个新的 `.value`给 __Ref__ ，同样新的数据也仍然是响应式的。
```typescript
interface Ref<T> {
  value: T
}

function ref<T>(value: T): Ref<UnwrapRef<T>>

```

### ShallowReactive与ShallowRef
除此以外，__Vue__ 还新增加了两种变体类型 __ShallowReactive__ 与 __ShallowRef__ ：
```typescript
function shallowRef<T>(value: T): ShallowRef<T>

function shallowReactive<T extends object>(target: T): T
```
顾名思义，__ShallowReactive__ 、 __ShallowRef__ 与前者不同的区别在于其不是`deep`转换的，因此：
```typescript
const state = shallowRef({ count: 1 })

// 不会触发更新
state.value.count = 2

// 会触发更新
state.value = { count: 2 }

const state = shallowReactive({
  foo: 1,
  nested: {
    bar: 2
  }
})

// 改变自身属性，会触发更新
state.foo++

// 内嵌对象不是响应式数据
isReactive(state.nested) // false

// 不会触发更新
state.nested.bar++
```
与前者相比 __ShallowReactive__ 与 __ShallowRef__ 对性能更好友好，更加适合一些不常更新的数据对象，以避免内存资源的过渡占用。

## 响应式数据是如何实现的？
前面我们介绍了在 __Vue.js 3.0__ 中的新的API与其功能，现在是时候来谈下其底层原理， __Vue__ 是如何来实现数据响应式处理的，首先我们将介绍一种非常重要的基础数据结构 __Dep__ 。
### Dep与数据收集 
在 __Vue.js__ 2.0与3.0中 __Dep__ 都是实现整个响应式数据的核心，__Dep__ 是一种如下的数据结构：
```typescript
export type Dep = Set<ReactiveEffect> & TrackedMarkers

type TrackedMarkers = {
  w: number // wasTracked: 数据被收集次数
  n: number // newTracked: 数据新增被收集次数
}
```
它很简单仅仅拥有两个属性`w: wasTracked`与`n: newTracked`，但却是整个响应式数据实现的根本。  
首先我们需要知道对于一个响应式数据更新而言，他需要有两个最基本的角色：观察器`Effect`与收集器`Dep`，每当有数据需要被更新时`Dep`负责收集更新信息，告知`Effect`，哪些数据需要被更新，而`Effect`则负责处理这些信息并最终完成视图的更新。  
那么我们首先来看`Dep`是如何被创建并完成收集的：
```typescript
// 第一步，在Getter中跟踪属性
track(target, TrackOpTypes.GET, key)

// 第二步，为每个属性创建一个Dep用于追踪收集
depsMap.set(key, (dep = createDep()))

// 第三步，判断属性是否需要被追踪，如需要则构建Dep与effect之前的双向绑定，从而完成Dep的创建与追踪收集
if (shouldTrack) {
  dep.add(activeEffect!)
  activeEffect!.deps.push(dep)
}
```
在完成了`Dep`的创建与收集之后，我们完成了响应式数据处理的第一步，即：收集数据，那么现在我们到了第二步：如何去监听数据的变化。

### Effect与视图更新
由于`Dep`的创建与收集是在 __Proxy Getter__ 中完成的，那么数据的变化监听自然则是在 __Proxy Setter__ 中完成。  
与 __Vue.js 2.0__ 使用 __Watcher__ 来实现数据监听不同，在 __Vue.js 3.0__ 中则是使用了 `Effect`来完成这个操作。首先我们来看下`Effect`是什么？
```typescript
class ReactiveEffect<T = any> {
  active = true
  deps: Dep[] = [] // Dep收集器
  parent: ReactiveEffect | undefined = undefined // 上级Effect作用域

  computed?: ComputedRefImpl<T> // 是否计算属性
  allowRecurse?: boolean

  onStop?: () => void //停用回调
  onTrack?: (event: DebuggerEvent) => void // 追踪回调
  onTrigger?: (event: DebuggerEvent) => void // 刷新回调

  constructor(
    public fn: () => T,
    public scheduler: EffectScheduler | null = null,
    scope?: EffectScope
  ) {
    recordEffectScope(this, scope) // 将Effect推入当前作用域Scope
  }

  run() {} // 检查是否需要更新视图

  stop() {} // 停止Effect运行
}
```
简单的来说`Effect`是一个收集器，它会收集`Dep`的数据更新信息，每当有属性需要被更新时，会触发 __Proxy Setter__ 钩子，然后检查需要被更新的数据`Dep`通知与当前`Dep`数据所绑定的`Effect`，`Effect`在接收到更新信息之后将会调用`scheduler`函数将此`EffectScope`范围内的视图加入到下一个更新队列中，从而完成响应式数据的视图更新。

### EffectScope
前面我们讲述了，`Dep`与`Effect`是如何分别在 __Proxy__ 的`Getter`与`Setter`中，完成相互的绑定与视图更新的，同时我们也提到了在 __Vue.js 3.0__ 中的一个新的属性 __EffectScope__，那么 __EffectScope__ 有什么用呢？  
在 __Vue.js 2.0__ 中 __Vue__ 对于任意的响应式数据(`data`)、计算属(`computed`)性、观察器(`watcher`)以及`component`组件都会构建一个`Watcher`观察器，对于数据、计算属性等的观察器而言，是用于监听数据的变化更新，而对于`component`组件而言这个观察器则是用于组件的视图更新。  
我们提到了在 __Vue.js 3.0__ 中 __Vue__ 使用了`Effect`来替换了以往的`Watcher`，而`EffectScope`则正是替换了以往的`component`组件的`Watcher`。不仅如此`EffectScope`现在提供了更为灵活的使用方式，以允许你构建一片区域，在这片区域中的任何`data`、`computed`、`watcher`等所构建的`efffect`都将得以独立运行。

```typescript
const scope = effectScope()

// 此区域内的所有Effect将会变成一个整体，统一独立处理
scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(doubled.value))

  watchEffect(() => console.log('Count: ', doubled.value))
})

// 停用当前Scope
scope.stop()
```

## 计算属性与观察器
与`Ref`和`Reactive`函数所构造的响应式数据相同，`Computed`计算属性与`Watch`观察属性本质上都是构建了一个自身的`Effect`对象，只是他们在使用上的用途不同。

### Computed计算属性
`Computed`计算属性：计算属性接收一个`Getter`函数然后返回一个仅可读的`Ref`对象。
```typescript
// read-only
function computed<T>(
  getter: () => T,
  // see "Computed Debugging" link below
  debuggerOptions?: DebuggerOptions
): Readonly<Ref<Readonly<T>>>
```

### Watch观察器
`Watch`观察器用于监听数据的变化，并执行相对应的回调函数，
```typescript
function watch<T>(
  source: WatchSource<T>,
  callback: WatchCallback<T>,
  options?: WatchOptions
): StopHandle

interface WatchOptions extends WatchEffectOptions {
  immediate?: boolean // default: false 是否立即触发回调
  deep?: boolean // default: false 是否深度监听
  flush?: 'pre' | 'post' | 'sync' // 回调执行顺序
  onTrack?: (event: DebuggerEvent) => void // 属性收集钩子函数
  onTrigger?: (event: DebuggerEvent) => void // 属性刷新钩子函数
}
```
与 __Vue.js 2.0__ 相比 __Vue.js 3.0__ 新增了`flush`属性与`onTrack`、`onTrigger`两个钩子函数，`flush`的三个属性 `'pre' | 'post' | 'sync'`分别对应`Watch`钩子函数在组件刷新前、中、后三个不同的时期运行，而`onTrack`与`onTrigger`则分别是数据收集与刷新时所要触发的钩子函数。

## 总结
得益于 __Vue.js 3.0__ 在 __Componsition Api__ 与 __Setup__ 上所带来的代码语法书写上的提升，__Vue__ 引入了全新的声明式函数 __Ref__ 以及 __Reactive__ 来满足新的语法需求。同时增加了 __shallowRef__ 与 __toRefs__ 之类的变体函数来丰富功能的多样式，  
在使用了新的 __ActiveEffect__ 来作为观察器替代了以往的 __Watcher__ 来作为数据监听之后，__Vue__ 新增了 __EffectScope__ 作用域来更进一步提升使用与性能。  
本章我们主要介绍了 __Vue.js 3.0__ 的 __Reactive__ 模块的功能以及是如何去实现数据的双向绑定的，下一章我们将详情介绍 __Vue.js__ 应用是如何去运行，以及组件是如何去更新的。





