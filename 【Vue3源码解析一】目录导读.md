## 开篇介绍
本篇文章是自己对于Vue3源码学习解读系列的第一篇文章。导读篇，本篇主要介绍Vue3源码各个模块的作用，以此作为后续系列文章的开头。由于此系列文章是基于自身对于Vue源码理解而写，若有误错之处，还望请指正包涵。

## 入口
Vue3.0采用模块化来设计整个源码的目录结构，packages目录下包含了Vue全部的功能模块。  
其中 __Vue__ 目录是整个项目的开始入口，__Vue__ 目录仅起到项目文件入口导出的作用，在 __Vue__ 中会直接导出整个 __Vue.js__ 中的全部 __API__ ，而具体的功能则包涵在 __Runtime-Core__  中。  
__Script__ 目录是项目的脚手架，起到项目开发时运行以及针对不同环境下需对 __Vue.js__ 进行针对性编译的功能。 

## Runtime-Dom
继续跟随前进的步伐，我们遇到了第二个模块 __Runtime-Dom__ ，与 __Vue__ 入口文件相同，__Runtime-Dom__ 同样也只是导出别的模块的功能。在这里Vue导出了应用的创建函数 __createApp__，此外 __Runtime-Dom__ 也导出了其自身对于全局应用的一些辅助功能，如：自定义HTML元素、指令帮助器。

``` typescript
// 应用创建函数
export const createApp = () => {} as CreateAppFunction<Element>

// 同时也导出了SSR的应用创建函数
export const createSSRApp = () => {} as CreateAppFunction<Element>

// 创建自定义HTML元素
export {
  defineCustomElement,
  defineSSRCustomElement,
  VueElement,
  VueElementConstructor
} from './apiCustomElement'

// 指令辅助功能以及内部指令
export {
  vModelText,
  vModelCheckbox,
  vModelRadio,
  vModelSelect,
  vModelDynamic
} from './directives/vModel'
export { withModifiers, withKeys } from './directives/vOn'
export { vShow } from './directives/vShow'

import { initVModelForSSR } from './directives/vModel'
import { initVShowForSSR } from './directives/vShow'
```

## Runtime-Core
__Runtime-Core__ 模块是我们遇到的第一个核心模块，__Runtime-Core__ 模块构建了 __Vue.js__ 应用自身大多的API功能，除了引入 __Reactivity__ 模块的数据响应式处理以外，__Runtime-Core__ 提供了最为核心的数据监听、计算属性功能，实现了数据层向UI层的实时映射。  
同时 __Runtime-Core__ 模块也提供了生命周期钩子函数、组件任务队列刷新、应用创建、Render函数等功能。

``` typescript
// 生命周期钩子函数
export {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onActivated,
  onDeactivated,
  onRenderTracked,
  onRenderTriggered,
  onErrorCaptured,
  onServerPrefetch
} from './apiLifecycle'
```

```typescript
// 组件应用创建
export function defineComponent<Props, RawBindings = object>(
  setup: (
    props: Readonly<Props>,
    ctx: SetupContext
  ) => RawBindings | RenderFunction
): DefineComponent<Props, RawBindings>
```

```typescript
// 计算属性与观察器
export { computed } from './apiComputed'
export {
  watch,
  watchEffect,
  watchPostEffect,
  watchSyncEffect
}
```

## Reactivity
__Reactivity__ 模块是 __Vue.js__ 的数据响应式模块，此模块实现了 __Vue.js__ 对于数据与UI视图的双向绑定，实现了 __Vue.js__ 对于数据以及组件等 __Wachter__ 观察器的构建从而实现了  __Vue.js__ 的组件更新机制。  
从 __Vue.js 3.0__ 开始 __Vue__ 对于数据响应式的实现由 __Observer API__ 替换为使用 __Proxy__ 来实现，同时整个 __Reactivity__ 模块也支持自定义构建。

```typescript
// 在Vue3.0中，响应式数据分类两类 Ref与 Reactive
declare const RefSymbol: unique symbol
export interface Ref<T = any> {
  value: T
  [RefSymbol]: true
}
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}

// 同时Reactive模块也提供了不同功能变体函数,如ShallowRef、shallowReactive等
export type ShallowRef<T = any> = Ref<T> & { [ShallowRefMarker]?: true }
export type ShallowReactive<T> = T & { [ShallowReactiveMarker]?: true }
```
 与 __Vue 2.0__ 相同响应式数据的实现原理同样是依靠 __Dep__ 这个核心的基础数据结构来实现，对于每一个在 __Vue__ 中的基础响应式数据，__Vue__ 都会在其中构建一个 __Dep__ 来收集数据更新的状态，同时在组件中构建一个 __Watcher__ 来监测组件的更新，依赖 __Dep__ 的消息推送，来构建组件的更新队列，最终完成对于UI视图的更新。

 ## Compiler
前面所提到的模块都在谈论 __Vue__ 应用自身的构建流程，而并没有谈论到模板编译的过程， __Compiler__ 模板则正是整个 __Vue__ 编译器的实现，对于任何一个框架而言，编译器无疑是最为核心的存在。

首先我们整个模板渲染的步骤：  

1. 由 __Runtime-Core__ 模块构建 __Vue__ 应用，解析HTML模板入口
2. 解析HTML模板，构建VNode节点，生成AST语法抽象树。
3. 将AST语法树进行代码优化，包括对于VNode的Diff算法的实现，数据结构优化。
4. 最终生成Render函数，导出可以构建UI视图的组件Render函数。

![image](https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/3.15d9566b.png)

然后我们简单看下实际代码模块中的作用：

### Runtime-Dom
``` typescript
// Runtime-Dom模块除了导出Runtime-Core模块中Compile功能以外，还包括了自身对于指令、CSS样式、事件等的解析功能
import {
  baseCompile,
  baseParse,
  CompilerOptions,
  CodegenResult,
  ParserOptions,
  RootNode,
  noopDirectiveTransform,
  NodeTransform,
  DirectiveTransform
} from '@vue/compiler-core'
import { parserOptions } from './parserOptions'
import { transformStyle } from './transforms/transformStyle'
import { transformVHtml } from './transforms/vHtml'
import { transformVText } from './transforms/vText'
import { transformModel } from './transforms/vModel'
import { transformOn } from './transforms/vOn'
import { transformShow } from './transforms/vShow'
import { warnTransitionChildren } from './transforms/warnTransitionChildren'
import { stringifyStatic } from './transforms/stringifyStatic'
import { ignoreSideEffectTags } from './transforms/ignoreSideEffectTags'
import { extend } from '@vue/shared'
```

### Runtime-Core
```typescript

```

## Shared
__Shared__ 模块是辅助函数模块，这里没有API功能。

## 总结
 __Vue.js 3.0__ 采用模块化的源码设计结构使得我们对于整个源码的理解更为简洁了然，同时模块化的代码也使得 __Vue.js 3.0__ 支持更为高度的自定义化。  
 在整篇文章中，我们整体上依照 __Vue.js__ 的运行机制，根据各个模块的运行顺序来不断深入，简明介绍 __Vue.js__ 的各个模块的功能，后续我们将更为详细的为各个模块做做出说明。

