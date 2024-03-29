## 前言
我们在前面的 __Reactivity响应式数据构建之路__ 与 __Vue.js的运行机制与生命周期__ 章节中均提到了 __Vue.js__ 的刷新机制`Scheduler`，  

虽然在源码中`Scheduler`只有不过几百行代码，但它却控制着整个 __Vue.js__ 应用的运转核心，接下来让我们来详细谈谈 `Scheduler`的作用。  

注意`Scheduler`仅仅是负责控制`task`何时更新，它并不执行任何实际的操作。

## 什么是Scheduler？
如果将整个 __Vue.js__ 应用比喻为一座工厂，那么`Scheduler`则是这座工厂的调度室，它负责整个 __Vue.js__ 应用的任务队列的管理与运行，  

同时也需要知道，`Scheduler`并不负责任何具体的任务实现，它仅仅是负责控制任务`task`何时执行以及其执行顺序。

<img align="center" width="500" src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/%E6%99%BA%E8%83%BD%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0%E7%B3%BB%E7%BB%9F.png" />

## Scheduler的三种类型
__Vue__ 拥有三种不同的`Scheduler`任务队列类型，它们分别是：
1. preFlushCbs 前置任务队列
2. flushCbs 同步任务队列
3. postFlushCbs 后置任务队列

每当一个新的轮询刷新任务开启，__Vue__ 首先会去 `preFlushCbs 前置任务队列`，然后去执行`flushCbs 同步任务队列`，最后再去执行`postFlushCbs 后置任务队列`。  

对于每一个任务队列而言，它都存在两种状态`waiting`与`flushing`即等待执行与执行中。每当一个新的`SchedulerJob`开始，__Vue__ 将会创建一个新的`micro task`微任务来执行这个`SchedulerJob`。  

顾名思义，`preFlushCbs`会在`SchedulerJob`也就是组件刷新之前执行，而`flushCbs`则是负责执行组件刷新，最后`postFlushCbs`会在组件刷新完成之后执行。  

__Vue__ 在`watchEffect api`中提供了三种不同的函数形式：`watchEffect`、`watchPostEffect`、`watchSyncEffect`也正是在此基础之上构建而来的。

那么这里我们首先提出一个问题为什么 __Vue__ 需要三种不同状态的`Scheduler`，而不是一种呢？我们先卖个关子，将答案放在后面揭晓。

## 如何开启一个新的任务队列?

### 任务堆栈
__Vue__ 通过`queueJob`开启一个新的`SchedulerJob`，这里首先会对任务进行去重校验，防止重复执行，校验完成之后将新的任务推入当前任务栈中，并开始执行。

```typescript
export function queueJob(job: SchedulerJob) {
  if (
    (!queue.length ||
      !queue.includes(
        job,
        isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex
      )) &&
    job !== currentPreFlushParentJob
  ) {
    if (job.id == null) {
      queue.push(job) // 将新任务推入任务栈
    } else {
      queue.splice(findInsertionIndex(job.id), 0, job) // 重新排列重复任务
    }
    queueFlush() // 执行下一个刷新任务
  }
}
```

### 开始执行

这里首先`isFlushPending`将会被设置为`true`，意味着任务即将开始执行。之后flushJobs将会被推入下一个微任务队列中，开始正式执行任务队列。

```typescript
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true // 开始等待执行
    currentFlushPromise = resolvedPromise.then(flushJobs) // 在下一个微任务中执行SchedulerJob
  }
}
```

## 为什么要使用resolvedPromise？

在这里有一个非常重要的细节，__Vue__ 使用了`Promise`来开启一个新的`SchedulerJob`任务队列执行，而并不是直接开始执行，为了了解这个原因，我们首先介绍一些别的东西：“js的事件轮询机制”

```typescript
const resolvedPromise: Promise<any> = Promise.resolve()
```

### js的事件轮询机制

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/asdasdas11251251.png" width="400" />

对于浏览器而言js的运行是单线程的，`script`、`html render`、`event`这类的任务它们属于其中的`macro task`宏任务，而`promise`这样的则是属于`micro task`微任务。  

微任务的执行顺序优先于宏任务，每一个宏任务之后都会紧跟一个微任务的执行，微任务的存在确保了浏览器的状态在其执行前后的一致性，因为微任务的执行会优先于`HTML Render`这样的宏任务。  

而 __Vue__ 正是巧妙的使用了这一规则，将每一个新的`SchedulerJob`通过`Promise`来推入下一个微任务队列，这样便可以使得其在组件刷新之前执行任务队列，同时也确保了`SchedulerJob`
任务队列之间的执行顺序，避免了不同任务队列之间执行顺序错乱的问题。


## 开始执行flushJobs

`flushJobs`函数将会正式开始执行 __Vue__ 的任务队列，正如前面所提到的，这里`preFlushCbs`、`flushCbs`、`postFlushCbs`将会依序执行，  

值得注意的是，__Vue__ 在这里将会将任务进行重新排序，以确保父级组件的执行顺序优先于子级组件，因为父级组件首先被构造，所以其拥有更高的优先级。

```typescript
function flushJobs(seen?: CountMap) {
  isFlushPending = false
  isFlushing = true
  if (__DEV__) {
    seen = seen || new Map()
  }

  flushPreFlushCbs(seen)

  // 确保父级组件的刷新优先于子级组件
  queue.sort((a, b) => getId(a) - getId(b))

  const check = __DEV__
    ? (job: SchedulerJob) => checkRecursiveUpdates(seen!, job)
    : NOOP

  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        if (__DEV__ && check(job)) {
          continue
        }
        // console.log(`running:`, job.id)
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = 0
    queue.length = 0

    flushPostFlushCbs(seen)

    isFlushing = false
    currentFlushPromise = null
    
    // 
    if (
      queue.length ||
      pendingPreFlushCbs.length ||
      pendingPostFlushCbs.length
    ) {
      flushJobs(seen)
    }
  }
}
```
## 执行pre前置任务

`pre前置任务`执行于组件刷新之前，它确保了被执行的函数的组件状态的一致性，前面我们所提到过的 __watch api__ 正是属于前置任务的一种，  

前置任务会被遍历递归执行，以确保所有的任务均被执行完毕。

```typescript
export function flushPreFlushCbs(
  seen?: CountMap,
  parentJob: SchedulerJob | null = null
) {
  if (pendingPreFlushCbs.length) {
    currentPreFlushParentJob = parentJob
    activePreFlushCbs = [...new Set(pendingPreFlushCbs)]
    pendingPreFlushCbs.length = 0
    if (__DEV__) {
      seen = seen || new Map()
    }
    for (
      preFlushIndex = 0;
      preFlushIndex < activePreFlushCbs.length;
      preFlushIndex++
    ) {
      if (
        __DEV__ &&
        checkRecursiveUpdates(seen!, activePreFlushCbs[preFlushIndex])
      ) {
        continue
      }
      activePreFlushCbs[preFlushIndex]()
    }
    activePreFlushCbs = null
    preFlushIndex = 0
    currentPreFlushParentJob = null
    // 递归执行，确保所有前置任务均被执行完毕
    flushPreFlushCbs(seen, parentJob)
  }
}
```

## 执行async同步任务

遍历执行任务，在这里便会去执行组件的刷新`component.update()`任务，同时 __Vue__ 会使用`try catch`来包裹整个流程以确保后序的后置任务得以执行。

```typescript
try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        if (__DEV__ && check(job)) {
          continue
        }
        // console.log(`running:`, job.id)
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  }
```

## 执行post后置任务

`post后置任务`主要是主要是处理一些`effect`效果，在这里已经完成了组件的刷新`component.update()`，同时`watch api`如果选择后置模式的话，也会在这里得以执行。

```typescript
export function flushPostFlushCbs(seen?: CountMap) {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)]
    pendingPostFlushCbs.length = 0

    // #1947 already has active queue, nested flushPostFlushCbs call
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    activePostFlushCbs = deduped
    if (__DEV__) {
      seen = seen || new Map()
    }

    activePostFlushCbs.sort((a, b) => getId(a) - getId(b))

    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      if (
        __DEV__ &&
        checkRecursiveUpdates(seen!, activePostFlushCbs[postFlushIndex])
      ) {
        continue
      }
      activePostFlushCbs[postFlushIndex]()
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```


## nextTick函数的本质
前面我们介绍了 __Vue__ 的`SchedulerJob`任务队列与微任务、宏任务之间的关系，由于 __Vue__ 的组件刷新会触发`HTML render`(HTML 重绘)，  

而这个操作是一个宏任务，我们知道微任务的执行将会紧跟在宏任务之后，因此对于`nextTick`而言其本质仅仅只是创建一个的`promise.then()`，  

将回调函数推入下一个微任务队列中，便可以保证此时函数的执行会在组件刷新完成之后。

## 总结
本章作为前一章 __Vue.js的运行机制与生命周期__ 的补充文章，详细的介绍了`Scheduler`模块是如何去运行的。  

理解`Scheduler`模块的核心在于理解为什么`Scheduler`具有`pre`、`sync`、`post`三种模式以及`Scheduler`与微任务、宏任务之间的关系。

__Vue__ 将每一个`instance.update()`组件刷新推入下一个`micro task`微任务，由于浏览器每执行一个宏任务之后都会立刻执行下一个微任务。

__Vue__ 巧妙的利用了这一规则以使得的组件刷新与任务执行堆栈之间不会冲突。

## 文献参考
[Microtasks](https://javascript.info/microtask-queue)  

[Event loop: microtasks and macrotasks](https://javascript.info/event-loop)  

[Vue3 task scheduler source code analysis](https://programs.wiki/wiki/vue3-task-scheduler-source-code-analysis.html)


