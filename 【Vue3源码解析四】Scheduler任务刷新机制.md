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
      queue.push(job)
    } else {
      queue.splice(findInsertionIndex(job.id), 0, job)
    }
    queueFlush()
  }
}
```

## 为什么要使用resolvedPromise？

## 开始执行flushJobs

### 执行pre前置任务

### 执行async同步任务

### 执行post后置任务



## nextTick的本质

## 总结
本章作为前一章 __Vue.js的运行机制与生命周期__ 的补充文章，详细的介绍了`Scheduler`模块是如何去运行的。  

理解`Scheduler`模块的核心在于理解为什么`Scheduler`具有`pre`、`sync`、`post`三种模式，__Vue__ 将每一个`instance.update()`组件刷新推入下一个`micro task`微任务，

由于浏览器每执行一个宏任务之后都会立刻执行下一个微任务， __Vue__ 巧妙的利用了这一规则以使得的组件刷新与任务执行堆栈之间不会冲突。


