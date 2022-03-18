## 前言

本章我们开始介绍 __Vue.js__ 的`Computed`计算属性与`Watch`观察器模块的源码，我们将在本章详细介绍这两个模块背后的原理。  

希望经过本篇文章的介绍能够让读者对于`Computed`与`Watch`的实现原理与实际使用更加深入。

## Computed计算属性

### computed函数

__Vue.js 3.0__ 使用`computed`函数来创建一个新的计算属性，`computed`接收`Get`、`Set`以及debugOptions参数，  

默认情况下`Set`将会为空，`Get`函数里面的响应式数据将会作为依赖被收集。

```typesscript
// computed
export function computed<T>(
  getter: ComputedGetter<T>,
  debugOptions?: DebuggerOptions
): ComputedRef<T>
```

### ComputedRefImpl




## Watch观察器

