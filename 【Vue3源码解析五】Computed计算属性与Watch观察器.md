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

`ComputedRefImpl`是`Computed`计算属性的实际控制器，每当创建一个新的计算属性`Computed`便会构造一个新的`ComputedRefImpl`类，  

首先`ComputedRefImpl`为每一个计算属性构造了一个新的`effect`，这个`effect`用于收集计算属性的依赖`dep`，  

每当计算属性被引用时，执行`trackRefValue`函数将`get`中所遇到响应式数据的`dep`与之前所创建好的`effect`收集器完成绑定，从而完成计算属性对于其内部响应式数据的收集。  

每当被收集的响应式数据更新时，`effect`会去执行`effect.run`操作将计算属性的更新推入下一个微任务更新队列中，更新视图。  

最后`dirty`属性将标识缓存本次计算所得到的结果，除非依赖被更新时`effect`上面的`scheduler`回调函数才会被执行`dirty`变回到`true`,  

开始重新计算计算属性的值。

```typescript
export class ComputedRefImpl<T> {
  public dep?: Dep = undefined

  private _value!: T
  public readonly effect: ReactiveEffect<T>

  public readonly __v_isRef = true
  public readonly [ReactiveFlags.IS_READONLY]: boolean

  public _dirty = true // 判断当前计算属性是否需要更新
  public _cacheable: boolean // 是否缓存数据

  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean,
    isSSR: boolean
  ) {
    // 创建新的effect
    this.effect = new ReactiveEffect(getter, () => {
      if (!this._dirty) {
        this._dirty = true // 依赖更新，重新计算计算属性值
        triggerRefValue(this) 
      }
    })
    this.effect.computed = this
    this.effect.active = this._cacheable = !isSSR
    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    // the computed ref may get wrapped by other proxies e.g. readonly() #3376
    const self = toRaw(this)
    trackRefValue(self) // 收集依赖
    if (self._dirty || !self._cacheable) {
      self._dirty = false
      self._value = self.effect.run()! // 执行计算属性更新操作
    }
    return self._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}
```




## Watch观察器

