## 前言
前面第二章，我们讲述了 __Reactive__ 响应式数据是如何去实现双向绑定的，但是却没有去叙述`Effect`最终是如何去完成视图更新的，  
这里我们将结合 __Vue.js__ 的生命周期，从应用构建开始去更加完整的讲述一个 __Vue.js__ 应用是如何完成构建并运行的。

## Vue.js应用的生命周期
首先，我们来看下 __Vue__ 官方所给出的生命周期示意图，这里我们将整个生命周期简单的分为四个阶段：
1. 应用构建阶段
2. 模板编译阶段
3. 运行阶段
4. 卸载阶段

本篇文章，我们将去讲述 __Vue.js__ 在应用构建阶段与运行阶段的一些操作与细节，以期为读者

<image src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/lifecycle.16e4c08e.png" align="center" width="80%" />



## 总结
下一篇，我们将开始介绍 __Vue__ 的 `Compile`编译器，来讲述 __Vue__ 是如何去完成模板编译的。
