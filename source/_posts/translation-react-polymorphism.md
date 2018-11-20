---
title: 【译文】React 的多态性
date: 2018-11-20 15:14:43
tags:
- React
- js底层
categories: 译文
---

> 原文链接 [Surprising polymorphism in React applications](https://medium.com/@bmeurer/surprising-polymorphism-in-react-applications-63015b50abc)
> 如有不当，欢迎指正！

## React 的多态性

现今，基于 [React](https://facebook.github.io/react/) 的 Web app 通常使用一些不可变的数据结构来管理状态，比如流行的 [Redux](http://redux.js.org/) 状态容器。 这种模式有几点好处，并且在 React/Redux 之外的领域愈加受欢迎。

这种机制的核心便是所谓的 `reducers`，它们是函数，根据触发的 `action` 更改某个状态到下一个状态————比如响应用户交互。有了这个核心的抽象概念，复杂的 `state` 和 `reducers` 可以组合成简化版本，方便了代码分段下的单元测试。请看下面来自于 [Redux 文档](http://redux.js.org/docs/basics/ExampleTodoList.html) 的例子：

```javascript
// redux-todo.js
const todo = (state = {}, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        id: action.id,
        text: action.text,
        completed: false
      }
    case 'TOGGLE_TODO':
      if (state.id !== action.id) {
        return state
      }

      return Object.assign({}, state, {
        completed: !state.completed
      })

    default:
      return state
  }
}
```

`todo` reducer 响应 `action` 将现存的 `state` 映射到一个新的 `state`。从性能优化的角度看代码，它似乎遵守了代码单态的原则，比如，保持对象的 `shape` 一致。

```javascript
// redux-todo-use.js
const s1 = todo({}, {
  type: 'ADD_TODO',
  id: 1,
  text: "Finish blog post"
});

const s2 = todo(s1, {
  type: 'TOGGLE_TODO',
  id: 1
});

function render(state) {
  return state.id + ": " + state.text;
}

render(s1);
render(s2);
render(s1);
render(s2);
```

坦白说，`render` 处理的属性应该是单态的，例如，`state` 对象的应当有一样的 `shape` —— [map or hidden class in V8 speak](https://github.com/v8/v8/wiki/Design%20Elements#fast-property-access)。在同一时刻，序列中的`s1` 和 `s2` 都有 `id` ，`test` ，`completed`  属性。然而，在 `d8` shell 上运行这段代码并追踪 ICs（inline cahces 内联缓存），我们观察到，`render` 侦测到不同的对象 shapes ，`state.id` 和 `state.test` 属性的访问是多态的。

![](https://cdn-images-1.medium.com/max/800/1*FrfEaOkxshIj79wJDQyrIQ.png)

那单态性从何说起？这其实很微妙，跟 V8 处理对象字面量的方式有关。每个对象字面量——比如，形如 `{a:va,...,z:vb}` 的表达式定义了 `transition` 树的根 `map` （记住 `map` 是 `V8 speak` 为 对象 `shape` 而生的）。如果你使用一个空的对象字面量 `{}` 作为 `transition` 树的跟，它就是不包含任何属性的 `map` ；如果使用 `{id:id, text:text, completed:completed}` 对象字面量，那么 `transition` 树的跟就是一个包含这些属性的 `map` 。看看一个简化的例子：

```javascript
// transition-tree.js
let a = {x:1, y:2, z:3};

let b = {};
b.x = 1;
b.y = 2;
b.z = 3;

console.log("a is", a);
console.log("b is", b);
console.log("a and b have same map:", %HaveSameMap(a, b));
```

你可以在 Node.js 上运行这段代码，并附上 `--allow-natives-syntax` 命令行标识（允许使用内置的 `%HaveSameMap` ）:

![](https://cdn-images-1.medium.com/max/800/1*yzSaH_AE5z7r9PWBXlvwWg.png)

尽管对象 `a` 和 `b` 看起来一样——拥有同样的属性，对应类型一样，顺序一样——它们的 map 却不同。这是因为它们有不同的 transition 树，如下图所示：

![](https://cdn-images-1.medium.com/max/800/1*fkbEgBWk74icFH1yZIH7Lw.png)

使用不同的（不相容）的对象字面量创建对象时，多态性便隐藏在其中。这点在常用的 `Object.assign` 中尤为显著，举个例子：

```javascript
// object-assign.js
let a = {x:1, y:2, z:3};

let b = Object.assign({}, a);

console.log("a is", a);
console.log("b is", b);
console.log("a and b have same map:", %HaveSameMap(a, b));
```

仍是产生的了不同的 map ，因为对象 `b` 始于一个空的对象（`{}` 字面量），`Object.assign` 只是把属性一个个拼连上去。

![](https://cdn-images-1.medium.com/max/800/1*Xu-nIj21gj-GlHDkzsSOSA.png)

这同样适用于有大量属性并通过 Babel 编译的场景。因为 Babel——或者其他编译器——在对处理大量属性的时候使用了 `Object.assign` 。

![](https://cdn-images-1.medium.com/max/800/1*F2x8lRcZ83pQDvftelFOgA.png)

避免多态性的一种方式是从始至终使用 `Object.assign` 方法，那么所有的对象就始于一个空的对象字面量。但在状态管理逻辑中，这可能成为一个性能瓶颈。

```javascript
// object-assign-everywhere.js
let a = Object.assign({}, {x:1, y:2, z:3});

let b = Object.assign({}, a);

console.log("a is", a);
console.log("b is", b);
console.log("a and b have same map:", %HaveSameMap(a, b));
```

也就是说，关于代码多态化的问题还未停歇。保持单态性，对你的大多数代码来说，或许根本不重要。在无所谓的优化之前，你应当细心衡量对比。
