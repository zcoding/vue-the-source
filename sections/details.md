# 一些细节

## 文件和目录结构

+ compiler 模板编译相关
    + compile-props.js 编译 prop
    + compile.js __主要__编译函数
    + index.js 导出
    + resolve-slots.js 编译 slot
    + transclude.js 编译 template 的相关函数
+ directives 指令相关
    + element 元素指令
        + index.js 导出
        + partial.js partial 指令
        + slot.js slot 指令
    + internal 内置的指令
        + class.js class 指令
        + component.js 组件指令
        + index.js 导出
        + prop.js prop 指令
        + style.js style 指令
        + transition.js transition 指令
    + public 公开的指令
        + model _v-model_
        + bind _v-bind_
        + cloak _v-cloak_
        + el _v-el_
        + for _v-for_
        + html _v-html_
        + if _v-if_
        + index.js 导出
        + on _v-on_
        + ref _v-ref_
        + show _v-show_
        + text _v-text_
    + priorities.js 指令的优先级定义
+ filters 过滤器相关
    + array-filters.js 数组过滤器
    + index.js 其它内置的过滤器
+ fragment 片段
+ instance 实例相关
    + api
        + data.js data 相关的实例方法
        + dom.js DOM 相关的实例方法
        + events.js 事件相关的实例方法
        + lifecycle.js 声明周期相关的实例方法
    + internal 初始化相关
        + events.js 事件初始化
        + init.js 初始化的入口函数
        + lifecycle.js 声明周期初始化
        + misc.js
        + state.js 数据相关初始化
 + vue.js Vue 类定义
+ observer Observer 类相关
    + array.js 对数组的特殊处理
    + dep.js Dep 依赖类
    + index.js __Observer__ 类，数据和依赖响应的具体实现方法
+ parsers 解析（指令等）相关
    + directive.js 解析指令
    + expression.js 解析表达式
    + path.js 解析路径
    + template.js 解析模板
    + text.js 解析文本
+ transition 过渡系统相关
    + index.js 导出
    + queue.js 过渡队列
    + transition.js Transition 类和方法定义
+ util 工具函数
    + component.js 组件相关
    + debug.js 调试相关
    + dom.js DOM 相关
    + evn.js 环境相关
    + index.js 导出
    + lang.js 语言相关
    + options.js 其它
+ batcher.js 任务处理机
+ cache.js 缓存系统
+ config.js 默认配置项
+ directive.js __Directive__ 类
+ global-api.js Vue 的类级 API
+ index.js 入口文件
+ watcher.js __Watcher__ 类

## Vue 的实例化过程

Vue 的实例化过程可以分为几个阶段：

1. 私有属性的初始化
2. 调用 `init` 钩子
3. 初始化数据监听
4. 初始化事件
5. 调用 `created` 钩子
6. 如果有 `el` 选项就挂载元素（开始编译）

### 私有属性的初始化

+ `this.$el = null`
+ `this.$parent = options.parent` 父组件
+ `this.$root = this.$parent ? this.$parent.$root : this` 根组件
+ `this.$children = []` 子组件
+ `this.$refs = {}` 子组件的引用
+ `this.$els = {}` DOM 元素的引用
+ `this._watchers = []` 监听器
+ `this._directives = []` 指令
+ `this._uid = uid++` 全局唯一的 ID
+ `this._isVue = true` Vue 标识
+ `this._events = {}` 事件监听
+ `this._isFragment` Fragment 标识
+ `_isCompiled, _isDestroyed, _isReady, _isAttached, _isBeingDestroyed, _vForRemoving` 生命周期相关
+ `this._context = options._context || this.$parent` 上下文
+ `this._scope = options._scope` 作用域
+ `this._data = {}`

### 初始化数据监听

+ 设置 `this.$data` 来代理访问 `this._data`
+ 初始化 props
+ 初始化 meta
+ 初始化 methods
+ 初始化 data
  - 将 `this._data` 转化为响应式数据
+ 初始化 computed
  - 每个 computed 都有独立的 getter 和 setter 并且绑定了 Watcher

### 初始化事件

+ 注册组件事件
+ 注册 options.events 和 options.watch

### 挂载元素

如果 `el` 选项已提供就调用 `this.$mount` 方法：

1. 如果已编译，直接返回（不能重复编译）
2. 找到挂载点元素，如果没有，就创建一个空的 div 作为挂载点
3. 调用编译方法 `this._compile`
  1. transclude
  2. 初始化 `this.el` 和 `this._fragment` 等，触发 `beforeCompile` 钩子
  3. 编译根节点 el 得到 link 函数
  4. 调用 link 函数
  5. 设置 unlink 函数
  6. 如果有 `replace` 就替换旧的元素
  7. 触发 `compiled` 钩子
4. 设置 `hook:attached` 和 `hook:detached` 事件监听
5. 触发 `attached` 事件，触发 `ready` 钩子

## LRU 缓存方案

为了提高时间效率，Vue 在编译、解析等多个地方使用了 LRU 缓存。

__LRU: Least Recently Used 最近最少使用算法__

Vue.js 使用的 LRU 算法使用了 [https://github.com/rsms/js-lru](https://github.com/rsms/js-lru) 的核心代码。

算法的核心思想是：

+ 维护一个双向链表，表尾指向最近使用的（最新的）项目，表头指向最近没使用的（最旧的）项目
+ 获取缓存项目时，如果该项目命中，则将该项目移到表尾
+ 设置缓存项目时，如果该项目命中，则将该项目移到表尾，否则直接将该项目添加到表尾
+ 当缓存满了，需要置换，将表尾元素移除，再设置新的项目

## nextTick 的实现

nextTick 就是一个异步函数，最简单也是最常见的实现方式就是 `setTimeout(0)`

Vue 的源码确实使用了`setTimeout`来实现，但是只是作为 fallback 的实现。实际上，Vue 优先使用了`MutationObserver`来实现。

```javascript
if (typeof MutationObserver !== 'undefined' && !hasMutationObserverBug) {
  var counter = 1
  var observer = new MutationObserver(nextTickHandler)
  var textNode = document.createTextNode(counter)
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = function () {
    counter = (counter + 1) % 2
    textNode.data = counter
  }
} else {
  // webpack attempts to inject a shim for setImmediate
  // if it is used as a global, so we have to work around that to
  // avoid bundling unnecessary code.
  const context = inBrowser
    ? window
    : typeof global !== 'undefined' ? global : {}
  timerFunc = context.setImmediate || setTimeout // 如果 setImmediate 可用，优先使用 setImmediate
}
```