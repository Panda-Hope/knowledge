## 什么是Vuex？
官网定义：
> Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用**集中式**存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

这段话说明vuex有两个特点：集中管理、状态变化可以预测。 在介绍这两个特点之前，先了解下什么是状态，以及为什么要对状态进行管理。

- **状态是驱动应用的数据源**。

  通俗的讲就是组件中绑定的数据。例如：导航控件绑定的菜单数据、控制导航展开或收拢的数据等等。


- **为什么要状态管理？**

  既然状态是组件需要绑定的数据，那为什么不直接写在组件的data方法里呢？当这个状态只在一个组件中使用时，当然直接在data中定义就可以。但是当多个组件都依赖同一个状态或者都会修改同一个状态时，处理起来就比较麻烦。

  因为在vue中，组件通过prop注册自定义属性。父组件通过给子组件的自定义属性赋值将数据传给子组件。另外子组件不能直接修改父组件的数据，它可以emit一个事件，将变更后的数据通过事件参数传给父组件，父组件捕获到事件后更新数据。 这样的话对下面这些场景处理起来就很麻烦：

    1. 两个兄弟关系（或者毫无关系）的组件都依赖于同一个状态，A组件修改状态后，B组件也要更新；此时A组件无法把修改后的状态传给B。
    2. 祖先向后代组件传递数据； 需要通过prop一层层的的传递，非常繁琐。

- **vuex**

  **集中管理**：vuex把组件的共享状态抽取出来，以一个全局单例模式进行管理，在这种模式下，我们的组件树构成了一个巨大的“视图”，不管在树的哪个位置，任何组件都能获取状态或者触发行为。

  **状态变化可以预测**：不允许组件直接修改状态，而是在vuex内部，通过mutation来修改状态，这样每次状态的变更都可以追踪；
  ![image](https://vuex.vuejs.org/vuex.png)

## Vuex的核心概念


每一个 Vuex 应用的核心就是 store（仓库），store集中管理所有的状态。它和单纯的全局对象比有两点不同：

1.  vuex中的状态是响应式的，若 store 中的状态发生变化，那么相应的组件也会相应地得到高效更新；
2.  组件不允许直接修改属于store的状态，而应执行 action 来分发 (dispatch) 事件通知 store 去改变，这样使得我们可以方便地跟踪每一个状态的变化；

创建store对象的时候需要传递构造器选项，具体如下所示。构造器选项有：state, mutations, actions, getters, modules等，下面是这些选项的用法。

```
const store = new Vuex.Store({
  state: {
    products: [{"id": 1, "title": "iPad 4 Mini", "price": 500.01, "inventory": 2},
        {"id": 2, "title": "H&M T-Shirt White", "price": 10.99, "inventory": 10},]
  },
  getters: {
    productsCount: state => {
        return state.products.length;
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})
```


- **State**

**如何在组件中展示状态？** 通过在vue根实例中注册store选项，store会注入到所有子组件中，这样子组件能通过this.$store访问到。
如果组件要显示一个状态，通常会先定义一个计算属性，然后在计算属性返回store中的状态，组件绑定该计算属性。

```
<template>
    <div>
        {{productsCount}}
    </div>
</template>

<script>
export default {
    computed: {
        productsCount() {
         return this.$store.state.productsCount;   
        }
    }
}
</script>
```
**mapState辅助函数**
当一个组件需要获取多个状态时，将这些状态都声明为计算属性会有些重复和冗余，mapState函数可以帮助我们少输入些代码。

例如: store中有两个状态， name, phone
```
 new Vuex.Store({
  state: {
    name: '张绍',
    phone: '13800138000'
  },
  ...
})
```
组件中通过mapState获取这两个状态（ ...是ES6中的扩展运算符， 具体用法可参考[《ES6 ... 扩展运算符》](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax))
```
<template>
    <div>
        {{name}} / {{phone}}
    </div>
</template>

<script>
import { mapState } from 'vuex'

export default {
    computed: {
        ...mapState([
          'phone',
          'name',
        ]),
    }
}
```


- **Getter**
  有时候我们组件需要显示加工处理后的状态数据，例如store中有一个待办任务列表的状态数据，但是组件需要显示已完成的待办任务，如果每个组件都在自己的计算属性进行过滤会重复操作，增加代码量。这个时候可以在store中定义一个getter， 在这个getter中进行计算过滤，然后组件的计算属性中直接返回这个getter。

```
const store = new Vuex.Store({
  state: {
    todos: [
      { id: 1, text: '...', done: true },
      { id: 2, text: '...', done: false }
    ]
  },
  getters: {
    doneTodos: state => {
      return state.todos.filter(todo => todo.done)
    }
  }
})
```
组件

```
computed: {
  doneTodosCount () {
    return this.$store.getters.doneTodosCount
  }
}
```

- **Mutation**
  不允许组件直接修改状态，更改状态的唯一方法是提交mutation，然后在mutation中修改；

在vuex.store的mutations选项中定义一个mutation, mutation回调函数接收一个参数state，通过state取到状态数据。

```
const store = new Vuex.Store({
  state: {
    count: 1
  },
  mutations: {
    increment (state) {
      // 变更状态
      state.count++
    }
  }
})
```
组件中提交mutation更改状态

```
<template>
    <el-button @click="incrementCount">修改状态</el-button>
</template>

<script>
export default {
    methods: {
        incrementCount() {
            this.$store.commit('increment');
        }
    }
}
</script>
```

- **Action**
  mutation中的操作是同步的，如果我们需要在异步执行某些操作后修改状态，则只能使用action。 Action可以包含异步操作，在Action中提交mutation来修改状态。

定义一个Action

```
const store = new Vuex.Store({
  ...
  actions: {
    increment (context) {
      setTimeout(() => {
          context.commit('increment')
        }, 1000)
    }
  }
})
```
组件中分发Action

```
this.$store.dispatch('increment')
```


- **Module**
  对于一个大型的应用，所有的状态都集中在一个对象中，会变得非常臃肿。vuex允许我们将store分割成模块，每个模块拥有自己的state, mutation, action, getter，甚至是嵌套子模块。类似下面代码：


```
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```


## Vuex的安装使用
1. 首先安装vuex

```
npm install vuex --save
```
2. 显式地通过Vue.use()，引入vuex

```
import Vuex from 'vuex'

Vue.use(Vuex)
```

3. 实例化Vuex.Store对象

```
const store = new Vuex.Store({
  state: {
    ...
  },
  getters: {
    ...
  },
  mutations: {
    ...
  }
  actions: {
      ...
  }
})
```
4. 将store注入到Vue实例中，这样该 store 实例会注入到根组件下的所有子组件中，且子组件能通过 this.$store 访问到

```
new Vue({
  el: '#app',
  router,
  store,
  template: '<App/>',
  components: { App },
});
```

注意事项：对于大型的应用，通常会把vuex相关代码分割到模块中，大概的项目结构如下：(官网有个[购物车](https://github.com/vuejs/vuex/tree/dev/examples/shopping-cart)的例子可以参考)
![image](https://note.youdao.com/yws/api/personal/file/DE62C649FAFE4DE891DF9AF656701B36?method=download&shareKey=c5335c54c025ea39551f5afa3ef41fb7)


## 参考资料
- [Vuex官方文档](https://vuex.vuejs.org/zh/)
- [Vue组件](https://cn.vuejs.org/v2/guide/components.html)
- [ES6 ... 扩展运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax)

## 后记
此文是对官网资料的学习总结，其中夹杂了些自己对vuex的理解，如果有理解有误的地方，欢迎大家指出。