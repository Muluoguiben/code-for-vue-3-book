### 第3章 Vue.js 3 的设计思路

#### 3.1 声明式地描述 UI

使用 **模板** 来 **声明式地描述UI**

- 使用与 HTML 标签一致的方式来描述 DOM 元素, div 标签 -> <div></div>
- 使用与 HTML 标签一致的方式来描述属性, <div id="app"></div>
- 使用 : 或者 v-bind 来描述动态绑定的属性, <div :id="dynamicId"></div> 
- 使用 @ 或者 v-on 来描述事件, <div @click="handler"></div>
- 使用与 HTML 标签一致的方式来描述层级结构, 例如一个具有 span 子节点的 div 标签 <div><span></span></div>

在 Vue.js 中, 哪怕是事件, 都有之对应的描述方式. **用户不需要手写任何命令式代码, 这就是所谓的声明式地描述UI.**



使用 JavaScript 对象 来描述

```js
	const title = {
        tag: 'h1',
        props: {
            onClick: handle
        },
        children: [
            { tag: 'span' }
        ]
    }
    
    
---------------------------------------
      <h1 @click="handle"><span></span></h1>
```



**使用 JS 对象描述UI会更加灵活**. **而使用 JS 对象来描述 UI 的方式, 其实就是所谓的虚拟 DOM**.

**正是因为虚拟 DOM 的这种灵活性, Vue.js 3 除了支持使用模板描述 UI 外, 还支持使用虚拟 DOM 来描述 UI.**

其实我们在 Vue.js 组件中手写的渲染函数就是使用虚拟 DOM 来描述 UI 的

```js
import { h } from 'vue'

export default {
	render() {
		return h('h1', { onClick: handler }) // 虚拟DOM
	}
}
```

**h 函数的返回值就是一个对象, 其作用是让我们编写虚拟 DOM 变得更加轻松**. h 函数就是一个辅助创建虚拟 DOM 的工具函数, 仅此而已

**Vue.js 会根据组件的 render 函数的返回值拿到虚拟 DOM , 然后就可以把组件的内容渲染出来了**.

#### 3.2 初识渲染器

虚拟DOM 就是用 JS 对象来描述真实的 DOM 结构.

h('div', 'hello')  ---渲染器---> 真实 DOM



```html
<body></body>

<script>

const vnode = {
  tag: 'div',
  props: {
    onClick: () => alert('hello')
  },
  children: 'click me'
}

function renderer(vnode, container) {
  // 使用 vnode.tag 作为标签名称创建 DOM 元素
	const el = document.createElement(vnode.tag)
  // 遍历 vnode.props 将属性、事件添加到 DOM 元素
  for (const key in vnode.props) {
    if (/^on/.test(key)) {
      // 如果 key 以 on 开头，那么说明它是事件
      el.addEventListener(
        key.substr(2).toLowerCase(), // 事件名称 onClick ---> click
        vnode.props[key] // 事件处理函数
      )
    }
  }

  // 处理 children
  if (typeof vnode.children === 'string') {
    // 如果 children 是字符串，说明是元素的文本子节点
    el.appendChild(document.createTextNode(vnode.children))
  } else if (Array.isArray(vnode.children)) {
    // 递归地调用 renderer 函数渲染子节点，使用当前元素 el 作为挂载点
    vnode.children.forEach(child => renderer(child, el))
  }

  // 将元素添加到挂载点下
  container.appendChild(el)
}

renderer(vnode, document.body) //  body 作为挂载点
</script>
```

`renderer(vnode, container)`

vnode: 虚拟 DOM 对象

container: 一个真实 DOM 元素, 作为挂载点, 渲染器会把虚拟 DOM 渲染到该挂载点上



renderer 的实现思路: 

- 创建元素: 把 vnode.tag 作为标签名称来创建 DOM 元素.
- 为元素添加属性和事件: 遍历 vnode.props 对象 -> 截取以 on 字符开头的事件, 小写化, 调用 addEventListener 绑定事件处理函数. 例如 onClick -> click, 调用 addEventListener
- 处理 children: 如果 children 是一个数组, 就递归调用 renderer 继续渲染, 此时要将刚刚创建的元素作为挂载点(父节点); 如果 children 是字符串, 则使用 createTextNode 函数创建一个文本节点, 并将其添加到新创建的元素内



#### 3.3 组件的本质

**组件的本质: 组件就是一组 (虚拟)DOM 元素的封装**.  

这组DOM 元素就是组件要渲染的内容.

我们可以定义一个**函数**来代表组件, 而函数的返回值就代表组件要渲染的内容

```html
<body></body>

<script>
    
const MyComponent = function () {
  return {
    tag: 'div',
    props: {
      onClick: () => alert('hello')
    },
    children: 'click me'
  }
}

const vnode = {
  tag: MyComponent
}

function renderer(vnode, container) {
  if (typeof vnode.tag === 'string') {
    // 说明 vnode 描述的是标签元素
    mountElement(vnode, container)
  } else if (typeof vnode.tag === 'function') {
    // 说明 vnode 描述的是组件
    mountComponent(vnode, container)
  }
}

function mountElement(vnode, container) {
  // 使用 vnode.tag 作为标签名称创建 DOM 元素
	const el = document.createElement(vnode.tag)
  // 遍历 vnode.props 将属性、事件添加到 DOM 元素
  for (const key in vnode.props) {
    if (/^on/.test(key)) {
      // 如果 key 以 on 开头，那么说明它是事件
      el.addEventListener(
        key.substr(2).toLowerCase(), // 事件名称 onClick ---> click
        vnode.props[key] // 事件处理函数
      )
    }
  }

  // 处理 children
  if (typeof vnode.children === 'string') {
    // 如果 children 是字符串，说明是元素的文本子节点
    el.appendChild(document.createTextNode(vnode.children))
  } else if (Array.isArray(vnode.children)) {
    // 递归地调用 renderer 函数渲染子节点，使用当前元素 el 作为挂载点
    vnode.children.forEach(child => renderer(child, el))
  }

  // 将元素添加到挂载点下
  container.appendChild(el)
}

function mountComponent(vnode, container) {
  // 调用组件函数，获取组件要渲染的内容（虚拟 DOM）
  const subtree = vnode.tag()
  // 递归调用 renderer 渲染 subtree
  renderer(subtree, container)
}

renderer(vnode, document.body)
</script>

```



使用一个 **JS 对象** 来表达组件

```html
<body></body>

<script>
    
const MyComponent = {
  render() {
    return {
      tag: 'div',
      props: {
        onClick: () => alert('hello')
      },
      children: 'click me'
    }
  }
}

const vnode = {
  tag: MyComponent
}
    
function renderer(vnode, container) {
  if (typeof vnode.tag === 'string') {
    // 说明 vnode 描述的是标签元素
    mountElement(vnode, container)
  } else if (typeof vnode.tag === 'object') {
    // 说明 vnode 描述的是组件
    mountComponent(vnode, container)
  }
}

function mountElement(vnode, container) {
  // 使用 vnode.tag 作为标签名称创建 DOM 元素
	const el = document.createElement(vnode.tag)
  // 遍历 vnode.props 将属性、事件添加到 DOM 元素
  for (const key in vnode.props) {
    if (/^on/.test(key)) {
      // 如果 key 以 on 开头，那么说明它是事件
      el.addEventListener(
        key.substr(2).toLowerCase(), // 事件名称 onClick ---> click
        vnode.props[key] // 事件处理函数
      )
    }
  }

  // 处理 children
  if (typeof vnode.children === 'string') {
    // 如果 children 是字符串，说明是元素的文本子节点
    el.appendChild(document.createTextNode(vnode.children))
  } else if (Array.isArray(vnode.children)) {
    // 递归地调用 renderer 函数渲染子节点，使用当前元素 el 作为挂载点
    vnode.children.forEach(child => renderer(child, el))
  }

  // 将元素添加到挂载点下
  container.appendChild(el)
}

function mountComponent(vnode, container) {
  // 调用组件函数，获取组件要渲染的内容（虚拟 DOM）
  const subtree = vnode.tag.render()
  // 递归调用 renderer 渲染 subtree
  renderer(subtree, container)
}

renderer(vnode, document.body)
</script>

```

上述代码中, vnode.tag 是表达组件的对象, 调用该对象的 render 函数得到组件要渲染的内容, 也就是虚拟 DOM

Vue.js 中的有状态组件就是使用对象结构来表达的.



#### 3.4 模板的工作原理

无论是手写虚拟 DOM(或者渲染函数) 还是使用**模板**(template), 都属于声明式地描述 UI.  模板 <=> **编译器**

编译器的作用其实就是将模板编译为渲染函数(虚拟DOM)

```js
<div @click="handler">
    click me
</div>

----------------------------
render() {
	return h('div', { onClick: handler}, 'click me')
}
```

以 vue 文件为例

```js
<template>
	<div @click="handler">
    	click me
    </div>
</template>

<script>
        export default {
			data(){},
            methods: {
                handler: () => {}
            }
}
</script>
```

其中 <template> 标签里的内容就是模板内容, 编译器会把模板内容编译为渲染函数并添加到 <script>标签块的组件对象上, 最终在浏览器中运行的代码就是: 

```js
export default {
			data(){},
            methods: {
                handler: () => {}
            },
    		render() {
                return h('div', { onClick: handler }, 'click me')
            }
}
```

**所以, 无论是使用模板, 还是直接手写渲染函数, 对于一个组件来说, 它要渲染的内容最终都是通过渲染函数产生的, 然后渲染器再把渲染函数返回的虚拟 DOM 渲染成为真实 DOM, 这就是模板的工作原理, 也就是 Vue.js 渲染页面的流程.**



#### 3.5 Vue.js 是各个模块组成的有机整体

**组件的实现依赖于渲染器, 模板的编译依赖于编译器,** 并且编译后生成的代码是根据渲染器和虚拟 DOM 的设计决定的, 因此 Vue.js 的各个模块之间是互相关联, 互相制约的, 共同构成一个有机整体. 

编译器识别初哪些是静态属性, 哪些是动态属性, 在生成代码的时候可以给渲染器附带一些信息.

`patchFlags`

渲染器看到这个属性时, 就知道只有这个虚拟DOM对象的属性是动态的, 就相当于省区了寻找变更点的工作量, 性能自然就提升了.

