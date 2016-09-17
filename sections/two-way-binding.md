# 双向绑定的基本原理

## 从 `Object.defineProperty` 开始说起

当我们在讨论 Vue 的时候，或者当我们讨论现代 MVVM 框架的实现方式的时候，常常会提到 `Object.defineProperty` 这个方法。我曾经在面试中，或者被面试中问到这个问题：Vue 是如何实现双向绑定的？大多数人给出的答案都是：利用了 `Object.defineProperty` 这个方法。

我也是大多数人之中的一个，所谓的“知其然不知其所以然”。

实际上，当我终于开始下定决心研究 Vue 的源码的时候，我才发现，原来 Vue 里面用到 `Object.defineProperty` 的地方有三个，而其中只有一个用来实现“双向绑定”，另外两个则起“代理”作用：

+ 1. 用 `this.$data` 代理 `this._data`

```javascript
Object.defineProperty(Vue.prototype, '$data', {
    get () {
      return this._data
    },
    set (newData) {
      if (newData !== this._data) {
        this._setData(newData)
      }
    }
  })
```

+ 2. 用 `this[name]` 代理 `this._data[name]`

```javascript
Object.defineProperty(self, key, {
  configurable: true,
  enumerable: true,
  get: function proxyGetter () {
    return self._data[key]
  },
  set: function proxySetter (val) {
    self._data[key] = val
  }
})
```

+ 3. 将普通数据转换为被观察对象，实现“响应式数据（reactive data）”，就是通过 getter/setter 的方法

```javascript
defineReactive (obj, key, val) {
  // 省略
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 省略
    },
    set: function reactiveSetter (newVal) {
      // 省略
      // 当 value 被设置时（如果新值 !== 旧值），就触发响应
    }
  }
}
```

第三个用法，就是实现“双向绑定”的关键之一：观察者模式。在 Vue 内部，有一个 `Observer` 类，用来实现观察者模式。

首先要明确一点，观察者模式是用来实现双向绑定中的“数据 -> 视图”这个方向的绑定。接下来，说一下 `Observer` 这个类的实现和作用。

## 关于 Observer 你需要知道的

观察者模式里面有两个对象：一个是观察者对象，就是 `Observer` 实例，另一个是被观察对象，就是 `vm._data` 对象。为了将两者关联起来，需要调用上面说到的 `defineReactive` 方法。

在这个方法内部，会为 `vm._data` 的“每一层级”的每一对象都创建一个 `Observer` 实例。例如，`vm._data` 在第一层级，其中 `vm._data.__ob__` 就是该对象的 `Observer` 实例；如果 `vm._data` 下面有一个对象 `vm._data.a` ，那么也会创建一个 `vm._data.a.__ob__` 对象与之对应。

Observer 类有两个关键成员：`this.dep` 和 `this.value` ，后者就是其观察的对象。例如， `vm._data.a.__ob__` 观察的就是 `vm._data.a` 这个对象。那 `this.dep` 是什么鬼？是 `Dep` 类的实例，即“依赖”。

“依赖”是“发布／订阅”模式的组成部分。在 Observer 中，依赖的作用有两方面：第一，当 getter 触发时，触发“发布／订阅”模式的“收集依赖（订阅）”过程；第二，当 setter 触发时，触发“发布／订阅”模式的“发布”过程。

这两个过程是如何进行的，接下来对“发布／订阅”模式和依赖进行分析。

## 发布／订阅

先来看一张图

![双向绑定原理图](../images/two-way-binding.jpg)

这张图清晰的讲解了双向绑定的原理。Observer 的作用仅仅是把 `vm._data` 进行了 getter/setter 转化，并没有决定当 `vm._data` 发生变化时应该如何更新视图（或者进行其他的响应），也就是说“数据 -> 视图”这个方向的绑定我们只完成了一半。那么剩下的一半如何处理？

所谓的视图更新，就是 DOM 操作，所以问题的答案应该在 DOM 模板中寻找。如果我们希望当 `vm._data` 发生变化时执行一些 DOM 操作，我们有两个办法：

1. 每次 `vm._data` 发生变化时都编译一次模板，然后才知道哪些节点需要更新
2. 预先编译模板，在编译的过程中，通过对特定规则的解析预先知道哪些节点需要更新以及如何更新，并且把这些更新操作与其依赖的 `vm._data` 对象关联起来，当 `vm._data` 发生变化时，根据关联关系__只触发__那些需要更新的节点的更新操作

Vue 的方案很明显是第二种。不过我想先说一下第一种方案，虽然它看起来是不可取的。如果我们的 `vm._data` 不是 getter/setter 而只是一个普通的“纯”对象，每次变化都需要重新编译模板，在编译的时候生成 virtual-dom 树，假设之前也有一棵 virtual-dom 树，那么我们可以通过 diff 算法得到补丁，然后进行 DOM 操作。这样做也是可取的。（我想 React 的做法大致应该是这样的）

那么说回 Vue 的做法。想要预先知道哪些节点需要更新以及如何更新，就要用到 Vue 的“指令”。

### 指令和编译

在 Vue 中，指令（Directive）都是“声明式（directive）”的，使用起来非常方便。不同类型的指令定义了不同的 DOM 操作方法，例如，`v-show` 指令可能会改变元素的 `style` 属性的 `display` 的值，`v-if` 指令则可能会将元素移出／移入当前 DOM 树，等等。这些规则都是预先定义好的。

在新建 Vue 实例的时候，如果提供了 `template` 选项，或者 `el` 选项，就可能触发 `compile` 方法，这个方法的主要工作就是解析指令。编译的过程比较复杂，我这里进行了简化，列出主要的步骤。

+ 1 如果给出了 `el` 选项，跳到第 2 步，否则，如果给出了 `template` 选项，就要先把字符串模板转为 DOM 模板
 + 1.1 通过 `document.createdocumentFragment` 创建一个 frag 对象
 + 1.2 通过 `document.createElement('div')` 创建一个 wrapper div 容器
 + 1.3 将 `template` 字符串作为 `innerHTML` 赋给 wrapper div
 + 1.4 复制 wrapper div 的子节点到 frag 对象
 + 1.5 跳到第 2 步
 + __有一些坑需要特殊处理：__ 对于 `tr`|`th`|`td` 这些元素（还有其它），不能直接作为 innerHTML 赋给 wrapper div ，因为这些元素必须在某些特定元素内存在，例如，`td` 必须被 `tr` 包裹。因此，需要在这些元素的外面包裹一层再赋给 wrapper div ，然后根据包裹的深度取子节点

+ 2 现在开始遍历以 `el` 为根节点的 DOM 子树（暂时忽略 DocumentFragment 的情况）
 + 2.1 调用 `compileNode` 方法（在后面再详细讲解），得到该节点的“link函数”，如果这个节点有子节点，则执行 2.2 否则 2.3
 + 2.2 调用 `compileNodeList` 方法：遍历每个子节点，对每个子节点调用 `compileNode` ，如果子节点还有子节点，则递归调用 `compileNodeList` ，最后调用 `makeChildLinkFn` 得到所有这些节点组成的“link函数”
 + 2.3 将 2.1 和 2.2 得到的“link函数”组合，得到根节点的“link”函数

+ 3 将“link”函数返回，就完成了 `el` 节点（以该节点为根节点的 DOM 子树）的编译

所以“link”函数是什么鬼？上述的步骤并没有真正把“模板（元素）”和“响应操作”关联起来，两者之间的关系需要调用“link”函数之后才算绑定在一起。这样做是因为，如果解析到某个元素就立即绑定，那么可能这个操作本身已经改变了 DOM 树，这就会影响了之前或之后编译的其他元素。而且，每个 link 函数调用完成之后，会生成一个对应的 unlink 函数，这个函数是用来解除绑定的，在清除监听器的时候非常重要。

前面只是整体的流程（还是简化的），关键的 `compileNode` 方法还没讲解。

compileNode 才是生成 link 函数的关键，因为 link 函数是将“指令”和“DOM 操作”关联起来的，而指令的解析在 compileNode 的过程中完成：

+ 1 区分节点类型，如果是 `ELEMENT_NODE` 跳到第 2 步，如果是 `TEXT_NODE` 跳到第 3 步，其它类型不处理

+ 2 编译一个 ELEMENT_NODE 并返回 link 函数
 + 如果是 `TEXTAREA` ，特殊处理：给它添加 `:value` 指令，指令值就是它的 `node.value` 解析出来的，然后再把 `node.value` 清空，接着继续处理
 + 如果有 `attributes` ，调用 `checkTerminalDirectives` 进行 terminal 指令编译（terminal 指令有两种，即 `v-for` 和 `v-if` ，所谓 terminal 指令，就是该指令会忽略掉在它之前的那些指令，这也是为什么要先编译 terminal 指令）
 + 如果目前还没有得到 link 函数，就调用 `checkElementDirectives` 进行元素指令编译
 + 如果目前还没有得到 link 函数，就调用 `checkComponent` 进行组件编译
 + 如果目前还没有得到 link 函数（并且存在 `attributes`），就调用 `compileDirectives` 进行普通指令的编译
 + 如果都没有得到 link 函数，只能返回 `undefined` （说明这个元素不需要编译，所以没有相关的 link 或者 unlink 函数）

+ 3 编译一个 TEXT_NODE 并返回 link 函数
 + 如果存在 `_skip` 标记，就跳过这个节点，直接返回 `removeText` 函数作为 link 函数（这个函数会把该节点去掉）
 + 调用 `parseText(node.wholeText)` 获取 tokens
 + 如果不存在 tokens 就返回 `null` （说明这个节点不需要编译）
 + 把当前节点的相邻文本节点都标记为 `_skip` （准备删掉）
 + 创建一个 `DocumentFragment` ，根据 `token.tag` 调用：如果是插值型 token 就调用 `processTextToken` ，如果是普通文本 token 就调用  `document.createTextNode` ，把生成的新节点添加到这个 fragment 里面
 + 以上步骤只是解析了文本节点的内容，还没有把插值和节点对应起来
 + 再调用 `makeTextNodeLinkFn` 创建并返回 link 函数

__简单的说，上述过程可以总结为：如果是元素节点，则通过编译元素的属性（如果有指令）／元素指令（如果是元素指令）／组件（如果是组件元素）得到 link 函数，否则通过编译文本节点的内容（如果有插值）得到 link 函数__

现在重点讲解__如何编译指令__和__如何编译插值__

指令在元素上使用时，需要指定指令名称、指令表达式、过滤器、参数等，而指令编译就是通过解析元素的属性字符串，抽取出这些指令组成部分。

解析完成之后，每条指令需要构建一个指令实例，也就是 `Directive` 类的实例。

`Directive` 实例的 `update` 方法定义了指令的更新方法，也就是 DOM 操作方法。那么，这个方法在什么时候调用呢？指令本身是不知道的，但是，__每个指令都维护一个 `Watcher` 实例 `this._watcher` ，这个对象知道什么时候需要执行更新方法，所以现在 DOM 操作的更新函数转交给 `Watcher` 来维护__。

__至于插值，其实会被转化为 `v-text` 或者 `v-html` 指令__，具体就不讲解了（原理就是正则匹配抽取插值部分，然后将原来的文本节删掉，并重新构建文本节点，这样就可以操作这些节点了）。

### Watcher: 订阅者

到目前为止，我们完成了：监听数据变化；编译节点，通过指令控制节点的更新方式。

那么，当数据变化时，如何通知指令更新节点？关键是 `Watcher` 类。

指令维护的是一个表达式，然而指令不知道这个表达式和 `vm._data` 有什么关系。例如，一个 `v-if="a > 1"` 指令，只知道 `a > 1` 这个表达式，却不知道它和 `vm._data.a` 的关系。但是 Watcher 知道！

__因为 Watcher 就是发布／订阅模式中的订阅者，它订阅了 vm._data 的变化，所以当变化发布时，它就可以执行更新方法，而它的更新方法就是指令的更新方法（的封装）__

如何订阅？需要借助于前面提到的一个重要的类：`Dep` 依赖

### Dep 连接 Watcher 和 Observer

Dep 即依赖，所谓依赖，就是“表达式（或函数）的值对 `vm._data` 的依赖”，也就是说，一个表达式（或函数）的值需要 `vm._data` 中的哪些属性（或嵌套属性）参与计算。

举个例子，表达式 `a.b` 依赖 `vm._data.a.b` 参与计算，即当 `vm._data.a.b` 发生变化时，表达式的值也要重新计算。

又例如，表达式 `a.b + a.c < 2` 依赖 `vm._data.a.b` 和 `vm._data.a.c` 参与计算，当两个被依赖的数据至少有一个发生变化时，表达式的值都要重新计算。

前面说到，指令的组成部分之一就是表达式，例如 `v-if="a.b + a.c < 2"` 这个指令的表达式就是 `a.b + a.c < 2` ，所以指令的值依赖 `vm._data.a.b` 和 `vm._data.a.c`

现在，我们终于可以把 DOM 更新和 `vm._data` 关联起来了：__在编译阶段，通过解析指令表达式，得出其依赖的响应式数据（这个过程会生成 Watcher 也就是订阅者），“依赖对象”由 Observer 对象维护（Observer 对象由 vm._data 维护），将依赖这个数据的订阅者添加到“依赖对象”的订阅者列表；当响应式数据发生变化时，由“依赖对象”通知订阅者列表的每个订阅者进行更新，然后和指令绑定的订阅者就进行 DOM 更新操作。__

这里还有一个关键点，源码中，依赖对象不是直接公开的（在闭包内），那怎么把订阅者添加到依赖的订阅者列表？

订阅者 Watcher 维护一个表达式，当 Watcher 实例创建的时候，这个表达式会计算一次，在计算的过程中，会触发响应式数据的 getter 方法。利用这一点，Vue 就是在 getter 方法执行时实现将“当前订阅者”添加到依赖的订阅者列表。这个过程也叫__“收集依赖”__。当然，并不是每次 getter 方法都会收集依赖，毕竟更多时候 getter 方法只需要返回值即可，所以需要用一个标识来辅助。具体到源码，这个标识就是 Dep.target 引用，每次 Watcher 在收集依赖之前都会把自己设置为“当前订阅者”，也就是 Dep.target 引用，然后再出发 getter 方法，这样，getter 方法内部判断 Dep.target 引用是否为空，如果不是，说明有订阅者正在收集依赖，那就让它收集，否则就直接返回值；然后，Watcher 收集完了自己的依赖，再把 Dep.target 引用设置为空。

至于通知的过程就更简单了，每个依赖对象有一个 `notify` 方法，当数据发生变化时，也就是 setter 方法触发的时候，其维护的依赖对象就会调用 `notify` 方法，这个方法会遍历其维护的订阅者列表，逐个通知，让他们调用自己的 `update` 方法，自己处理更新逻辑。

__至此，双向绑定的“数据 -> 视图”方向已经全部完成__

## “视图 -> 数据”方向的绑定

这个就简单多了。如果我们希望视图发生变化的时候做点什么，唯一可行的办法就是通过事件。例如，一个简单的 `<input>` 表单，如果我们希望在用户输入的时候做点什么，唯一可行的就是给这个元素添加 DOM 事件（可能是 input 或者 keydown/keyup 等事件）。

所以，要实现“视图 -> 数据”方向的绑定，只需要绑定事件即可。具体实现可以参考 `v-model` 指令的实现源码。
