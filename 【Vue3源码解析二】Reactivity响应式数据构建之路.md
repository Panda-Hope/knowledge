## Reactive模块的作用
本章我们将讲述在 __Vue.js__ 中 __Reactive__ 响应式数据是如何去构建的，与 __Vue.js 2.0__ 相比 __Vue.js 3.0__ 在底层仍然使用通过 __Dep__ 进行数据收集依赖更新的方式来完成响应式数据处理。  
但与之不同的是 __Vue.js 3.0__ 新增了 __Ref__ 数据结构，并使用了 __ReactiveEffect__ 来替换以往使用 __Watcher__ 来监听并更新数据的方式，同时增加了新的 __effectScope__ 作用域来提升性能。  
接下来让我们详细介绍 __Vue.js 3.0__ 所带来的新的响应式数据处理变化。

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

```


### EffectScope


## 计算属性与观察器
### Computed
### WatchEffect

## 总结
得益于 __Vue.js 3.0__ 在 __Componsition Api__ 与 __Setup__ 上所带来的代码语法书写上的提升，__Vue__ 引入了全新的 __Ref__ 数据结构，来满足新的语法需求。同时增加了 __shallowRef__ 与 __toRefs__ 之类的变体函数来丰富功能的多样式，  
在使用了新的 __ActiveEffect__ 来作为观察器替代了以往的 __Watcher__ 来作为数据监听之后，__Vue__ 得以与 __EffectScope__ 来做一个结合从而得到一个新的性能上的优化提升。





