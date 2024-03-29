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

开始重新执行计算属性的结果值。

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

### Watch观察器的组成部分

在 __Vue.js 3.0__ 中`watch`观察器存在`watchEffect`与`watch`函数两种，而其本质均是`doWatch`的变体函数。 

一个完整的`watch`观察器主要包括两个部分：

- Getter：被观察的属性，在watchEffect中是回调函数
- SchedulerJob：被执行的回调函数

### getter

getter存在三种不同情形，当被观察的是一个响应式数据`ref`或者`reactive`时，其返回的是被观察的响应式数据本身，  

而当需要同时观察多个数据时，其返回的则是被观察的响应式数据的一维数据，与计算属性相同，这里的数据同样会被依赖收集。  

最后当被观察的是一个函数时，其返回的则是一个被`callWithErrorHandling`包裹的高阶函数。

```typescript
if (isRef(source)) {
  getter = () => source.value
  forceTrigger = isShallow(source)
} else if (isReactive(source)) {
  getter = () => source
  deep = true
} else if (isArray(source)) {
  getter = () => {...}
} else if (isFunction(source)) {
  getter = () => callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER)
}
```

### watchJob

在`watch`观察器中`getter`属于被观察的对象，而`watchJob`则是执行函数，与计算属性相同`watch`观察器中同样会构建`effect`用于依赖收集与组件更新。  

`watchJob`的执行分为三种情况，当`immediate`属性存在时`watchJob`会被当做独立函数立即执行，当依赖被更新或者组件更新时，`effect`将会被运行。  

而当`flush === 'post'`时，这个运行会被延后到组件刷新后的下一个微任务队列中。  

当`watchJob`被执行时同样会分为两种情况，当被观察的是响应式数据时，其与`computed`计算属性相同会收集响应式的`dep`依赖，当依赖更新时才会被触发，  

而当被观察的是函数时，这个`effect`会被关联到当前的`effectscope`作用域中，每当组件或者`effectscope`需要更新时，开始执行。

```typescript
const job: SchedulerJob = () => {
    if (!effect.active) {
      return
    }
    if (cb) {
      // watch(source, cb)
      const newValue = effect.run()
      if (
        deep ||
        forceTrigger ||
        (isMultiSource
          ? (newValue as any[]).some((v, i) =>
              hasChanged(v, (oldValue as any[])[i])
            )
          : hasChanged(newValue, oldValue)) ||
        (__COMPAT__ &&
          isArray(newValue) &&
          isCompatEnabled(DeprecationTypes.WATCH_ARRAY, instance))
      ) {
        // cleanup before running cb again
        if (cleanup) {
          cleanup()
        }
        callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
          newValue,
          // pass undefined as the old value when it's changed for the first time
          oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
          onCleanup
        ])
        oldValue = newValue
      }
    } else {
      // watchEffect
      effect.run()
    }
  }
```

### stopWatcher

`watch`函数会返回一个`stop`函数，当执行`stop`时，`watch`会销毁这个`effect`并从当前的`effectScope`中移除。

```typescript
return () => {
  effect.stop()
  if (instance && instance.scope) {
    remove(instance.scope.effects!, effect)
  }
}
```

## 总结

本章我们详细的介绍了`Computed`计算属性与`Watch`观察器的运行机制。  

本章内容并不复杂，读懂本章的核心仍然是在理解`effect`与`schduler`这两个模块，`Computed`与`Watch`均是在此基础上而来的。  

对于`effect`与`schduler`模块请阅读本系列文章的对应章节。



