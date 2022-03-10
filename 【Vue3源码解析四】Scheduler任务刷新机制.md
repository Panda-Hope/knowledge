## 前言
我们在前面的 __Reactivity响应式数据构建之路__ 与 __Vue.js的运行机制与生命周期__ 章节中均提到了 __Vue.js__ 的刷新机制`Scheduler`，  

虽然在源码中`Scheduler`只有不过几百行代码，但它却控制着整个 __Vue.js__ 应用的运转核心，接下来让我们来详细谈谈 `Scheduler`的作用。  

注意`Scheduler`仅仅是负责控制`task`何时更新，它并不执行的组件实际更新。

## Scheduler
