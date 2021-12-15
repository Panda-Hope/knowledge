# 深入理解 Vue3 的响应式

## 0 前言

随着 Vue3 的登台，各大博客论坛铺天盖地的涌来各种文章。我们组也是率先把 Vue3 应用到了工作中，刮起了一波学习热潮。上个月组内一位大佬的分享也让我受益良多，那么就借着这波余热一起从 Vue3 的源码一探究竟。

## 1 ref 和 reactive 的关系

Vue3 的文档中主要提供了两种比较常用的方法来把你的数据转换成响应式，`ref` 和 `reactive`。这两种方法的区别就在于 `ref` 在 `reactive` 的基础上对数据进行了二次分装，需要以 `.value`的形式获取值，可以从源码中看到：

```javascript
class RefImpl<T> {
  private _value: T

  public readonly __v_isRef = true

  constructor(private _rawValue: T, public readonly _shallow = false) {
    this._value = _shallow ? _rawValue : convert(_rawValue)
  }

  get value() {
    track(toRaw(this), TrackOpTypes.GET, 'value')
    return this._value
  }

  set value(newVal) {
    if (hasChanged(toRaw(newVal), this._rawValue)) {
      this._rawValue = newVal
      this._value = this._shallow ? newVal : convert(newVal)
      trigger(toRaw(this), TriggerOpTypes.SET, 'value', newVal)
    }
  }
}
```

```javascript
const convert = <T extends unknown>(val: T): T =>
  isObject(val) ? reactive(val) : val
```

重点在于 `convert` 方法，能看出最后还是会通过 `reactive` 方法来转换成响应式。也知道了因为 Proxy 不能监听简单数据类型所以 Vue 在基础数据类型上又加了一层对象来实现响应式。



## 2. reactive 做了什么

既然 `ref` 只是 `reactive`的一层伪装，那我们就直接看看 `reactive` 做了什么吧：

```javascript
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers
  )
}
```

在 `reactive` 方法中，Vue 首先判断了一下如果 target 是只读对象那么直接返回，否则创建一个响应式对象。

```javascript
if (
  target[ReactiveFlags.RAW] &&
  !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
) {
  return target
}

...

// only a whitelist of value types can be observed.
const targetType = getTargetType(target)
if (targetType === TargetType.INVALID) {
  return target
}
const proxy = new Proxy(
  target,
  targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
)
proxyMap.set(target, proxy)
```

然后就进入了 `createReactiveObject` 方法，代码很清晰，先判断了是否已经绑定过，没有就绑定。最后，通过判断目标的类型，如果是普通的对象 Object 或 Array，处理器对象就使用 `baseHandlers`，如果是 Set, Map, WeakMap, WeakSet 中的一个，就使用 `collectionHandlers`。

这里有一个小知识点，为什么 Vue 存储使用的是 WeakMap ？

```javascript
export const reactiveMap = new WeakMap<Target, any>()
```

WeakMap 的键值**必须是对象**，而且 WeakMap 的键值是不可枚举的，是弱引用。方便垃圾回收。

弱引用对应的当然就是强引用了，强引用有一种计数机制。举个例子当一个对象被创建时，计数为1，每有一个变量引用该对象，计数都会加1，只有当计数为0时，对象才会被垃圾回收。所以一旦造成循环引用等问题，就会造成内存泄漏。而 WeakMap 当它的键所指对象没有被其他地方引用时，就会被垃圾回收了。

## 3. baseHandlers

 `baseHandlers` 针对不同类型的响应式做了不同的处理，我们重点来看一下完全响应式 mutableHandlers 是如何处理的，完全响应式中 Vue 一共定义了五种方法：

- `get`
- `set`
- `deleteProperty`
- `has`
- `ownKeys`

从 Vue2 中我们知道，如果 Vue 想要在数据更新时通知界面渲染那肯定要收集依赖，而在数据被修改后触发依赖，那么我们就知道：get、has、ownKeys 是收集依赖，set、deleteProperty 是触发依赖。先看一下 get 方法中是如何收集依赖的。

```javascript
const res = Reflect.get(target, key, receiver)

if (
  isSymbol(key)
    ? builtInSymbols.has(key as symbol)
    : key === `__proto__` || key === `__v_isRef`
) {
  return res
}

if (!isReadonly) {
  // 只读不需要收集
  track(target, TrackOpTypes.GET, key)
}

if (shallow) {
  // 浅层响应，直接返回
  return res
}

if (isRef(res)) {
  const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
  return shouldUnwrap ? res.value : res
}

if (isObject(res)) {
  // 如果是深层对象需要递归绑定响应
  return isReadonly ? readonly(res) : reactive(res)
}
```
代码并不难理解，加了点注释。Vue 根据不同情况分别做了不同的处理方式，那么这是对象类型的，数组类型也做了针对性的处理。

```javascript
if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
  return Reflect.get(arrayInstrumentations, key, receiver)
}
```

针对数组通过 `arrayInstrumentations` 方法来收集数组的依赖。我们来看看 `arrayInstrumentations` 方法做了什么：

```javacscript
const arrayInstrumentations: Record<string, Function> = {}
// instrument identity-sensitive Array methods to account for possible reactive
// values
;(['includes', 'indexOf', 'lastIndexOf'] as const).forEach(key => {
  const method = Array.prototype[key] as any
  arrayInstrumentations[key] = function(this: unknown[], ...args: unknown[]) {
    const arr = toRaw(this)
    for (let i = 0, l = this.length; i < l; i++) {
      track(arr, TrackOpTypes.GET, i + '')
    }
    // we run the method using the original args first (which may be reactive)
    const res = method.apply(arr, args)
    if (res === -1 || res === false) {
      // if that didn't work, run it again using raw values.
      return method.apply(arr, args.map(toRaw))
    } else {
      return res
    }
  }
})
```
在这里 Vue 通过封装了数组方法来达到收集依赖的目的。

再回到之前的 get 方法中，Vue 使用了 `Reflect.get`。Reflect 是一个内置对象，它与 Proxy 几乎共用方法。那为什么 Vue 要多此一举用 Reflect 呢？ 我们来看个例子：

```javascript
const data = new Proxy([1,2,3], {
    get(target, key) {
        console.log('get', key)
        return target[key];
    },
    set(target, key, value) {
        console.log('set', key, value)
        return true;
    }
})
data.push(4)
```

这是一个普通写法，`set` 方法为什么要返回一个 true，如果不返回 true 则会抛出一个 typeError。看上去是不是不太优雅，那么我们用 Reflect 改造一下。

```javascript
const data = new Proxy([1, 2, 3], {
  get(target, key, receiver) {
    console.log('get', key);
    return Reflect.get(target, key, receiver);
  },
  set(target, key, value, receiver) {
    console.log('set', key, value);
    return Reflect.set(target, key, value, receiver);
  },
});
data.push(4);
```

改造完后，我们来看看打印结果，打印出来是这样的：

```javascript
get push
get length
set 3 4
set length 4
```

这样达到的效果是一样的，不过还是有点问题。你会发现打印了两次 get 和 set，这是因为在 push 后，不仅改变了数组内的元素，同时操作了数组的 length 产生了两次 get 和 set，那么 Vue 是如何解决这种重复问题的？

```javascript
const hadKey =
  isArray(target) && isIntegerKey(key)
    ? Number(key) < target.length
    : hasOwn(target, key)
const result = Reflect.set(target, key, value, receiver)
// don't trigger if target is something up in the prototype chain of original
if (target === toRaw(receiver)) {
  if (!hadKey) {
    // key 不存在说明是新增操作，触发依赖
    trigger(target, TriggerOpTypes.ADD, key, value)
  } else if (hasChanged(value, oldValue)) {
    // 新旧值不相等，触发依赖
    // 同时也防止了刚才提到的 push 后触发了两次依赖的问题，新旧值相同则不触发依赖
    trigger(target, TriggerOpTypes.SET, key, value, oldValue)
  }
}
return result;
```

在 set 方法中，Vue 通过判断 key 是否为 target 自身属性，以及设置 val 是否跟 target[key] 相等来阻止重复的响应。

继续看 set 和 get 方法中，用到了 trigger 和 track 方法。看一下源码中他们都来自于一个叫 effect 的文件，那么 effect 是做什么用的？

```javascript
export function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}
```
其中创建 effect 的其实是 `createReactiveEffect` 方法，我们来看看这个方法：

```javascript
function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(): unknown {
    if (!effect.active) {
      return options.scheduler ? undefined : fn()
    }
    if (!effectStack.includes(effect)) {
      // 清除依赖
      cleanup(effect)
      try {
        enableTracking()
        effectStack.push(effect)
        // 
        activeEffect = effect
        return fn()
      } finally {
        effectStack.pop()
        resetTracking()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect
}
```
在 `createReactiveEffect` 方法中，首先对 effect 做了一些初始化，然后初次创建 effect 的时候如果当前的 effect 栈（effectStack）不包含当前 effect，那么清空之前的依赖，避免依赖的重复收集。然后执行传进来的 `fn` 方法，`fn` 方法是 vm 原型上的一个叫做 `componentEffect `。执行该方法后也就进入了渲染流程了。在这里，effect 最终赋值给了 activeEffect 中。

那我们先继续来看看 trigger 和 track 做了什么：

```javascript
export function track(target: object, type: TrackOpTypes, key: unknown) {
  // activeEffect 为空代表没有依赖，直接返回
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  // targetMap 是一个 WeakMap，用于收集依赖
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    // 如果目标没有被追踪，添加一个
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    // 如果目标 key 没有被追踪，添加一个
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    // 如果没有添加 activeEffect，添加一个
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    ...
  }
}
```
在 track 方法中，主要追踪并收集了依赖，再来看看 trigger：

```javascript
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  // 没有收集依赖的话直接返回
  if (!depsMap) {
    // never been tracked
    return
  }

  // 依赖队列，这里的依赖队列是 Set 类型避免重复
  const effects = new Set<ReactiveEffect>()
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  if (type === TriggerOpTypes.CLEAR) {
    ...
  } else if (key === 'length' && isArray(target)) {
  	// 这里就能监听到数组的 length 变化了
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= (newValue as number)) {
        add(dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      add(depsMap.get(key))
    }

    // 在 新增/删除/编辑 的方法中，判断了 target 的类型然后添加 depsMap 中的不同依赖到 effect 中
    switch (type) {
      case TriggerOpTypes.ADD:
        ...

      case TriggerOpTypes.DELETE:
        ...
      case TriggerOpTypes.SET:
        ...
    }
  }

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) {
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

  // 触发依赖
  effects.forEach(run)
}
```

在 trigger 方法中，拿到了之前收集到的依赖（也就是之前添加好的 effect）并添加到了任务队列中。然后遍历触发依赖。

其实已经有 Vue2 内味儿了，其实就是一个改良版的发布订阅模式。get 时通过 track 收集依赖，而 set 时通过 trigger 触发了依赖，而 effect 收集了这些依赖并进行追踪，在响应后去触发相应的依赖。effect 也正是 Vue3 响应式的核心。

了解了这些后，那么剩下的三个方法就很明朗了，一看就能明白：

```javascript
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}

function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}

function ownKeys(target: object): (string | number | symbol)[] {
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}
```

那么 Vue3 的响应式相关的比较核心的源码基本就是这些了，至于 `collectionHandlers` 方法，基本是换汤不换药，核心也是通过 trigger 和 track 实现，有兴趣的大佬可以自己看看。

## 4 参考

* [MDN Rlect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect)
* [Vue3 中文文档](https://www.vue3js.cn/docs/zh)
* [Vue3 源码](https://github.com/vuejs/vue-next)