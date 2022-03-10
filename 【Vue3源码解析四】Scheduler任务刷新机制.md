## 前言
我们在前面的 __Reactivity响应式数据构建之路__ 与 __Vue.js的运行机制与生命周期__ 章节中均提到了 __Vue.js__ 的刷新机制`Scheduler`，  

虽然在源码中`Scheduler`只有不过几百行代码，但它却控制着整个 __Vue.js__ 应用的运转核心，接下来让我们来详细谈谈 `Scheduler`的作用。  

注意`Scheduler`仅仅是负责控制`task`何时更新，它并不执行的组件实际更新。

## 什么是Scheduler？
如果将整个 __Vue.js__ 应用比喻为一座工厂，那么`Scheduler`则是这座工厂的调度室，它负责整个 __Vue.js__ 应用的任务队列的管理与运行，  
同时也需要知道，`Scheduler`并不负责任何具体的任务实现，它仅仅是负责控制任务`task`何时执行以及其执行顺序。

<img align="center" width="500" src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/%E6%99%BA%E8%83%BD%E8%B0%83%E5%BA%A6%E5%B9%B3%E5%8F%B0%E7%B3%BB%E7%BB%9F.png" />

## Scheduler的三种类型

