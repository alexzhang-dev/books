# 第 3 章 Vue.js 3 的设计思路

## 1. 声明式的描述 UI

Vue 采用了与 HTML 相近的形式声明式的描述 UI。

- 使用与 HTML 相同的标签元素，例如 `<div>`、`<p>` 等
- 使用与 HTML 一致的方式来描述元素的属性，例如 `id="foo"`、`class="bar"`
- 使用 `:` 或 `v-bind`来描述元素的动态属性，例如 `:id="foo"`、`v-bind:class="bar"`
- 使用 `@` 或 `v-on` 来描述元素的事件，例如 `@click="handleClick"`

当然你可以使用 JS 对象来描述元素

```ts
const title = {
  tag: 'div',
  props: {
    onClick: handleClick
  },
  children: "hello"
}
```

转换为模板描述 UI 是这样的

```html
<div @click="handleClick">hello</div>
```

在 Vue 可以使用 `h` 函数来辅助创建虚拟 DOM 节点，如果一个组件，Vue 将通过组件的 `render` 方法的返回值来确定组件的虚拟 DOM。

```ts
export {
	render() {
		return h("div", { onClick: handleClick }, "hello")
	}
}
```

## 2. 初识渲染器

虚拟 DOM 其实是 JS 对象来描述真实 DOM 结构的抽象。那么该如何将虚拟 DOM 渲染为真实的 DOM 呢？这就要依赖于 Vue 的渲染了。

渲染器分为两大块：创建与更新

### 创建

在创建元素阶段，渲染器做的工作其实并不复杂：

- 根据 tag 创建元素
- 根据 props 向元素中挂载属性或事件
- 处理 children，如果 children 是一个数组，那么就递归处理 children，如果 children 是一个字符串，那么就创建一个 TextNode

### 更新

而在更新阶段，需要做的事情就有分为两大部分：找到差异，更新差异。

找到这一个阶段非常复杂，主要是通过 Vue 的 diff 算法来找到最小差异。

## 3. 组件的本质

虚拟 DOM 除了可以描述真实 DOM 节点之外，还可以用来描述组件。

```ts
// 组件
const Foo = {
  render() {
    return h('div', null, "foo")
  }
}

// main
const main = {
  render() {
    return h(Foo)
  }
}
```

组件的本质其实就是一组 DOM 节点的集合。我们可以定义一个函数或一个 Object 来描述一个组件，该函数或者该对象的 `render` 的返回值就是这个组件需要渲染的 DOM 元素集合。例如我们知道 h 函数的第一个参数也就是虚拟节点的 tag 可以描述其对应真实节点的元素名。我们可以进行判断

- 如果说该虚拟节点的 tag 是普通的字符串，那么说明其虚拟节点描述的是一个真实的节点
- 如果说该虚拟节点的 tag 是一个函数或者对象，那么说明其虚拟节点描述的是一个组件

所以渲染组件其实和渲染普通的元素原理基本相同，只不过渲染组件需要先通过 `render` 来获取到其需要渲染的虚拟节点集合。

## 4. 模板的工作原理

上面我们说了如何通过渲染器将虚拟 DOM 渲染为真实的 DOM，下面我们再来说一说如何将模板转为虚拟节点。

```html
<div>hello</div>
```

编译为

```ts
render() {
  return h('div', null, 'hello')
}
```

最终通过渲染器渲染。编译器是一个很大的话题，我们这里就一笔带过，后面再说。

## 5. Vue.js 是各个模块组成的有机整体

我们知道，组件的渲染是通过渲染器实现的，而模板的编译则是通过编译器实现的，Vue 中的各个模块相互关联，相互制约。那么编译器与渲染器是如何协作的呢？

我们来看一个例子：

```html
<div id="foo" :class="cls"></div>
```

该模板通过编译器最终会编译为这样：

```ts
render() {
  return h('div', null, { id: 'foo', class: cls })
}
```

我们知道，渲染器在更新阶段的主要工作就是找到变化的点，但是这样每一次都要去遍历整个节点然后去找，那么效率肯定是很低的，而编译器在编译阶段，如果发现了某些数据是不会变的，那么在渲染器更新阶段时就会很轻松。例如

```ts
render() {
  return h('div', null, { id: 'foo', class: cls, patchFlag: 1})
}
```

- 编译器可以在编译时找到不会变的数据，例如这里的 id
- 然后给编译好的加上一个标记例如，`patchFlag`，可能这里的 `1` 代表的就是 id 属性不变
- 那么在更新阶段就不必去对比新旧的 class 了。性能自然就会提升



