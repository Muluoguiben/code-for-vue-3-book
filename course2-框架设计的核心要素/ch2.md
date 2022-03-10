### 第2章 框架设计的核心要素



##### 2.1 框架提供友好的警告信息 console.warn 函数

##### 2.2 控制框架代码的体积 

开发环境时  `__DEV__` 为 true

生产环境时 `__DEV__` 为 false 

一段分支代码永远都不会执行, 永远不会执行的代码称为 dead code, 它不会出现在最终产物中, 在构建资源的时候就会被移除(tree-shaking)

**框架在开发环境中为用户提供友好的警告信息的同时, 不会增加生产环境代码的体积**



##### 2.3 框架要做到**良好**的 Tree-Shaking

**简单来说, Tree-Shaking 指的就是消除那些永远不会被执行的代码, 也就是 dead code.**

**想要实现 Tree-Shaking, 必须满足一个条件, 即模块必须是ESM, 因为 Tree-Shaking 依赖 ESM 的静态结构**



**Tree-Shaking 另一个关键点 —— 副作用** . **如果一个函数调用会产生副作用, 那么就不能将其移除**

简单地说, 副作用就是, 当调用函数的时候会对外部产生影响, 例如修改了全局变量. 

(Vue中) 如果 obj 对象是一个通过 Proxy 创建的代理对象, 那么当我们读取对象属性时, 就会出发代理对象的 get 夹子(trap), 在 get 夹子中是可能产生副作用的, 例如我们在 get 夹子中修改了某个全局变量.



`/*#__PURE__*/` 作用就是告诉构建工具(rollup.js / webpack) , 对于某个函数的调用不会产生副作用, 可以放心地对其进行 Tree-Shaking.



##### 2.4 框架应该输出怎样的构建产物

`__DEV__`

`process.env.NODE_ENV !== "production"`

**用户可以通过 webpack 配置自行决定构建资源的目标环境**



##### 2.5 特性开关

在设计框架时, 框架会给用户提供诸多特性.

例如我们提供 A 、 B 、C 三个特性给用户, 同时还提供了 a b c 三个对应的特性开关, 用户可以通过设置 a b c 为 true 或者 false 来代表开启或者关闭对应的特性, 这将会带来很多溢出

- 对于用户关闭的特性, 我们可以利用 Tree-shaking 机制让其不包含在最终的资源中
- 该机制为框架设计带来了灵活性, 可以通过特性开关任意为框架添加新的特性, 而不用担心资源体积变大. 同时, 在框架升级时, 我们也可以通过特性开关来支持遗留API, 这样新用户可以选择不使用遗留API, 从而使最终打包的资源体积最小化

例如 `__Vue__OPTIONS__API`

**Vue3 仍然兼容 Options API 来编写代码. 但是如果明确知道自己不会使用 Options API, 用户就可以使用 `__VUE_OPTIONS_API__`开关来关闭该特性, 这样在打包的时候 Vue.js 的这部分代码就不会包含在最终的资源中, 从而减小资源体积**

```js
// webpack.DefinePlugin 插件实现
new webpack.DefinePlugin({
    __Vue_OPTIONS_API__: JSON.stringify(true) // 开启特性
})
```



##### 2.6 错误处理

`claaWithErrorHandling`函数

在Vue.js中, 我们也可以注册统一的错误处理函数

```js
import App from 'App.vue'
const app = createApp(App)
app.config.errorHandler = () => {
    // 错误处理程序
}
```



##### 2.7 良好的 TypeScript 类型支持

```tsx
function foo<T extends any>(val:T): T {
    return val
}
const res = foo('str')
// 此时 res: "str" 类型

---------------------------------------
function foo(val: any) {
    return val
}
const res = foo('str')
// res: any 类型
```

`runtime-core/src/apiDefineComponent.ts`

**使用TS编写代码框架和框架对TS类型支持友好是两件完全不同的事**. 有时候为了让框架提供更加友好的类型支持, 甚至要花费比实现框架功能本身更多的时间和精力