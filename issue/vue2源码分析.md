### 前言
该文章，对vue2.x原理和源码进行分析；本文拆分为了如下的八个章节：
```typescript
  1、准备工作：介绍了Flow、Vue.js的源码目录设计、Vue.js的源码构建方式，以及从入口开始分析了Vue.js的初始化过程；
  2、数据驱动：详细讲解了模版数据到Dom渲染的过程，从 new Vue开始分析了：mount、render、update、patch等流程；
  3、组件化：分析了组件化的实现原理，并且分析了组件周边的原理实现，包括：合并配置、生命周期、组件注册、异步组件等实现；
  4、深入响应式原理：详细讲解了数据的变化如何驱动试图的变化，分析了响应式对象的创建，依赖收集、派发更新的实现过程，一些特殊情况的处理，并对比了计算属性(compute)和侦听属性的实现(watch)，最后分析了组件更新的过程；
  5、编译：从编译的入口函数开始，分析了编译的三个核心流程的实现：parse -> optimize -> codegen；
  6、扩展：详细讲解了event、v-model、slot、keep-alive、transition、transition-group 等常用功能的原理实现，该章节用于分析Vue提供的各种特性；
  7、Vue-Router：分析了Vue-Router的实现原理，从路由注册开始分析了路由对象、matcher，并深入分析了整个路径切换的实现过程和细节；
  8、VueX：分析了Vuex的实现原理，深入分析了它的初始化过程，常用API以及插件部分的实现。
```

### Part1、准备工作
> 从第一章开始，我们将分析Vue源代码，首先会先介绍一些前置知识如flow、源码目录、构建方式、编译入口等。

一、Flow
- 由于动态类型语言Javascript很灵活，导致能编译通过的代码可能在运行的时候会有各种奇怪的bug，为了给动态语言添加约束，Vue源码使用了Flow工具；
- Flow 是facebook出品的JavaScript静态类型检查工具，Vue.js源码是用Flow做了静态类型检查，即在编译的时候发现类型错误，从而保证项目的可维护性、代码健壮和可读性。

Flow 的两个工作方式
```typescript
  1、类型推断：Flow会通过变量的上下文来推断出变量的类型，根据推断来决定检查是否通过；
    /*@flow*/
    function splitStr(str) {
      return str.split('')
    }
    splitStr(123) // Flow会检查出number类型不可以进行split从而编译时报错

  2、类型注释，我们在较复杂场景还是应该告诉Flow各个变量的类型，添加类似ts的类型注释可以提供Flow更明确的检查依据；
    /*@flow*/
    function add (x: number, y: number): number {
      return x + y
    }
    add('aaa', 123) // Flow会检查出'aaa'不是number

    /*@flow*/
    var foo: ?string = null // 检查通过，类型前添加'?'可以让这个变量赋值为 null 或者 undefined
```

二、目录结构，本文主要研究 vue.js 的src目录，src又分为了六个子目录；
并且各个目录按EsModule将功能拆分成模块，相关的逻辑放在一个独立的目录下维护，把复用的、通用的代码也放在一个目录。
这样的目录设计让代码的阅读和可读性变强，是非常值得学习和推敲的，下面介绍六个核心目录：

- compiler
```typescript
  1、compiler 目录包含 Vue.js 所有编译相关的代码。它包括把模版解析为 Ast 语法树、Ast语法树优化、代码生成等功能。
  编译的工作可以在构建时做（借助webpack、vue-loader）；也可以在运行时候做，使用包含构建功能的Vue.js。显然，编译是一项耗性能的工作，所以更推荐前者（离线编译）。
  2、vue2.0 的一个重要特性是 VirtualDom ，这个虚拟Dom的生成实际上是执行了 render 函数， 而我们平时写Vue大多是写template模版（在某些一句话调用组件的场景才会必须使用render函数)，那么模版到render函数的相关逻辑就在compiler目录。
```

- core
```typescript
  core 包含了 Vue.js 的核心代码，包括内置组件如keep-alive、全局API封装、Vue实例化、观察者、虚拟Dom、工具函数等；
  是Vue的核心代码。
```

- platform
```typescript
  Vue.js 是一个跨平台的MVVM框架，可以跑在 web 上，也可以配合weex跑在native客户端上；platform是 Vue.js 的一个入口，即可以让Vue分别打包在web、weex或者后来的mpvue平台上。
  本文会重点分析web入口打包后的Vue.js, 对于其他入口的打包暂不探究。
```

- server
```typescript
  Vue.js 2.0 支持了服务端渲染，所有相关的逻辑在这个目录下。
  服务端渲染主要的工作是把组件渲染为服务器端的 HTML 字符串，然后将他们直接发送到浏览器，最后将 HTML 字符串和客户端的静态标记“混合”为客户端上完全交互的应用程序。
```

- sfc
```typescript
  通常我们开发 Vue.js 会借助webpack构建，然后通过.vue单文件来编写组件。
  这个目录下的代码逻辑会把 .vue 这样的单文件组件 解析为一个 Javascript 对象
```

- shared
```typescript
  Vue.js 会定义一些工具方法，会被浏览器端的Vue.js 和server目录下的服务端Vue.js共享。
```

三、Vue.js 源码构建
- Vue.js 源码是基于 Rollup 构建，它的构建相关配置写在了 scripts 目录下。

- 我们通常会配置 `script` 字段作为NPM的执行脚本，Vue.js 也如此，它将构建的信息写在了 `package.json` 里：
  如下的三条命令的作用都是构建 Vue.js，他们的区别是后两条添加了一些环境参数，用于判断当前构建的具体平台；
  我们可以使用 `Node.js` 去获取到配置文件的参数如：const params = process.argv[2] // 'web-server-renderer'

  ```typescript
  {
    "script": {
      "build": "node scripts/build.js",
      "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
      "build:weex": "npm run build -- weex"
    }
  }
  ```
- 接下来我们沿着 package.json 找到并分析 `scripts/build.js` 的入口文件，首先我们看见了：
  ```typescript
    let builds = require('./config').getAllBuilds() // 读取所有“配置”信息
  ```

  build.js 从 `config` 里引入了所有的配置, 我们进入这个文件看下`配置`长啥样
  ```typescript
  // alias 返回一个对象，里面存放了 config.js的父级目录相邻的src下的六个核心目录的各入口文件的绝对路径信息（一个Map）
  // alias 就是提供了由 配置文件信息  -> src真实文件 的一个别名映射对象，可以快速获取对应真实文件如：由配置src/web/entry-compiler.js 的绝对路径
  const aliases = require('./alias')
  const resolve = p => {
    const base = p.split('/')[0]
    if (aliases[base]) { // 若别名Map中有这个base，则返回Map中的value的绝对路径，并拼凑当前入参的后缀
      return path.resolve(aliases[base], p.slice(base.length + 1))
    } else { // 若别名Map中没有这个base，则直接返回上级目录中的入参p，即可能从/dist、/src等目录寻找
      return path.resolve(__dirname, '../', p)
    }
  }
  const builds = { 
    // 可见config是向外暴露了关于：入口/出口等构建信息的一个对象；
    // getAllBuilds方法用于获取所有配置的一个对象数组；而这个方法获取的对象数组也符合rollup打包规范；
    // 即 getAllBuilds 方法将当前配置表通过映射和处理，生成了rollup所需要的配置数组

    'web-runtime-cjs-dev': {
      entry: resolve('web/entry-runtime.js'),
      dest: resolve('dist/vue.runtime.common.dev.js'),
      format: 'cjs',
      env: 'development',
      banner
    }
    // ...
  }
  if (process.env.TARGET) {
    module.exports = genConfig(process.env.TARGET)
  } else {
    exports.getBuild = genConfig
    exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
  }
  ```

  回到 build.js
  ```typescript
    if (process.argv[2]) { // 若是配置文件带了参数，则处理参数，从配置表中过滤出当前参数的配置
      const filters = process.argv[2].split(',')
      builds = builds.filter(b => {
        return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
      })
    } else { // 不带参数的build 默认是走web端
      builds = builds.filter(b => {
        return b.output.file.indexOf('weex') === -1
      })
    }

    // 获取好打包配置后，后续的操作就是遍历配置数组，然后链式调用去打包配置文件，最后writeFile输出到硬盘上。
  ```

  - Runtime Only Vs (Runtime + Compiler)
  这两个版本通常我们用 `vue-cli` 去初始化项目的时候会需要选择，他们的区别（主要是：是否在运行时具备编译模版的能力）如下：
  ```typescript
    Runtime Only：
      这个版本的js，即我们常用的.vue文件，通常需要借助如webpack的vue-loader来将.vue文件编译为Javascript，由于这个版本的文件不需要考虑在运行的时候编译.vue，所以它只包含运行时代码（runtime only），所以代码体积较轻量，也是我们推荐常用的（避免模版编译发生在运行时，造成性能损失）。

    Runtime + Compiler：
      若我们没有对代码做预编译，但又使用了 Vue 的template属性传入一个字符串，这个时候我们需要在客户端编译模版：
      // 不需要编译的版本, 因为实际上.vue文件在Runtime Only版本里，也是通过vue-loader在编译期，转化模版为了render函数
      new Vue({
        render (h) {
          return h('div', this.hi)
        }
      })
      // 需要编译的版本，在Vue2.0中，最终的渲染都是通过render函数来完成，如果写template属性，则需要编译成render函数；这个过程发生在运行时，则会有性能消耗
      new Vue({
        template: '<div>{{ hi }}</div>'
      })
  ```

  - Vue.js 构建总结
  ```typescript
    Vue.js是如何构建源代码的？
      答：Vue 通过 rollup 打包源代码；
      1、首先，Vue 在 scripts/config.js 中，写好了一套配置表Map，作为全量的配置表；
      2、然后，在打包脚本的入口文件：build.js源码中，Vue使用Node的能力去读取命令行参数，然后将用这些参数去全量配置表中筛选需要的配置；若命令行未传参，默认走浏览器端的Vue打包，即去除weex；
        if (process.argv[2]) {
          // 若配置文件带了参数，则处理参数，从配置表中过滤出符合当前参数的配置
          const filters = process.argv[2].split(',')
          builds = builds.filter(b => {
            return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
          })
        } else {
          // 不带参数的build，默认走web端的构建
          // filter out weex builds by default
          builds = builds.filter(b => {
            return b.output.file.indexOf('weex') === -1
          })
        }
      3、这些配置经过处理转化成了符合rollup规范的对象；在build.js中，收集完配置，就遍历筛选后的配置对象数组，链式调用进行相应配置文件的打包。 
  ```

  四、分析 `Vue.js` 构建的入口文件
  - 刚才分析了Vue的构建过程，现在分析一下Vue暴露出的打包入口文件，即Vue的初始化过程。
  实际上我们看到，Vue的核心就是一个用 Function 实现的类，我们只能通过 new Vue 去实例化它。然后通过Mixin，又给Vue挂载了很多原型上的方法；通过initGlobalAPI又给Vue添加了很多静态方法，以便后续的调用。

  - 以 `Runtime + Compiler` 的Vue版本为例分析
    首先我们先看我们引入的vue到底是什么，我们在node-modules里看package.json的main可以找到`dist/vue.runtime.common.js`文件，而由vue源码的配置表可以看到这个出口文件的对应入口文件为： `web/entry-runtime-with-compiler` ，我们沿着该文件向下寻找，可以找到Vue 的真实面目在 `core/instance/index.js`里；
    它实际上就是一个函数作为了构造函数，这样的好处是：可以方便后面的各种方法继续去拓展类的功能，而不是所有能力全定义在一个class里，这样的好处是便于维护拓展；
    ```typescript
      // core/instance/index.js
      function Vue (options) {
        if (process.env.NODE_ENV !== 'production' && 
          !(this instanceof Vue)
        ) {
          // Vue 是一个构造函数，需要结合new使用，即const a = new Vue({})；而不是直接const a = Vue() 将其当作普通函数；
          warn('Vue is a constructor and should be called with the `new` keyword')
        }
        this._init(options)
      }
      // 给Vue的原型上添加方法，这些函数都是和组件实例相关的
      initMixin(Vue)
      stateMixin(Vue)
      eventsMixin(Vue)
      lifecycleMixin(Vue)
      renderMixin(Vue)
      export default Vue
    ```
    这些Mixin 都是在Vue的 `prototype` 添加方法，这些方法是和组件实例相关；
    可见Vue按功能将这些 `能力拓展` 给拆分到多个模块里实现，这也是用函数作为构造函数的好处；

  - 在外层，Vue由对这个暴露出去的`Vue`对象做了更多能力拓展封装，如在 `initGlobalAPI(Vue)`中，Vue拓展了静态方法；
  ```typescript
    import Vue from './instance/index'
    import { initGlobalAPI } from './global-api/index'
    initGlobalAPI(Vue) // 实现了如：Vue.nextTick = nextTick等；
    // ...
    export default Vue
  ```

  Vue入口的调用关系：
  
    `web/entry-runtime-with-compiler.js`(加工Vue，添加解析器compile) 

    -> `web/runtime/index.js`（加工Vue，添加全局静态配置；原型上添加$mount、_patch_) 

    -> `core/index.js`（加工Vue，添加全局静态方法） 

    -> `core/instance/index.js`（暴露出Vue构造函数，在执行多个Mixin，给Vue原型添加方法）

  那么vue中为什么有的定义为静态方法，有的挂载到原型上呢？
  ```typescript
    1、直接挂到 Vue 的静态方法，通常是全局API，如 Vue.use 注册全局插件、Vue.component 注册全局组件，这些API与实例无关；
    2、挂载到 Vue.prototype 的方法，是和组件实例相关的；
    3、$ 开头的，一般是Vue实例提供的属性/函数，与用户定义的属性/函数区分开。
  ```


  ### Part2、数据驱动
  > 第二章开始，我们将分析Vue的数据驱动思想。所谓数据驱动，是指视图由数据的驱动所生成。

  我们修改视图的方式是通过修改数据，而不是去操作Dom。这样的代码只需要关心数据的修改，逻辑更加清晰（不用担心影响交互），这样的代码也是利于维护的。
  我们这一章，主要目的是分析：`数据和模版如何渲染为最终的Dom的`；而数据的修改引起视图的修改，是放在了后面的`深入响应式原理`章节。

  - 1、`new Vue` 发生了什么？

    还是回到 Vue的定义，可见Vue 是一个function实现的类，在 `new Vue` 实例的时候，其实最后会执行其原型上的 `_init` 方法，而这个方法的挂载就实现在下面的 `initMixin(Vue)`;

    ```typescript
      function Vue (options) {
        if (process.env.NODE_ENV !== 'production' &&
          !(this instanceof Vue)
        ) {
          warn('Vue is a constructor and should be called with the `new` keyword')
        }
        // 初始化 发生了什么？
        // 配置合并、初始化事件中心、初始化render渲染、初始化data、props、computed、watcher...
        this._init(options)
      }
    ```

    然后进入 `src/core/instance/init.js` 看见_init方法的挂载
    ```typescript
      export function initMixin(Vue: Class<Component>) {
        Vue.prototype._init = function (options?: Object) {
          const vm: Component = this
          // ...  -> 此处省略非主线代码
          // 配置合并
          if (options && options._isComponent) {
            initInternalComponent(vm, options)
          } else {
            vm.$options = mergeOptions(
              resolveConstructorOptions(vm.constructor),
              options || {},
              vm
            )
          }
          // 初始化 - 生命周期
          initLifecycle(vm)
          // 初始化 - 事件中心
          initEvents(vm)
          // 初始化 - render渲染
          initRender(vm)
          // 进入 beforeCreate 阶段
          callHook(vm, 'beforeCreate')
          // 初始化 - data
          initInjections(vm)
          initState(vm)
          initProvide(vm)
          // 进入 created 阶段
          callHook(vm, 'created')

          // 初始化的最后，如果检测到有 el 属性，则调用vm.$mount方法把实例挂载到dom上，目的就是：将模版渲染为Dom；
          if (vm.$options.el) {
            vm.$mount(vm.$options.el)
          }
        }
      } 
    ```

  - 可以看到，`new Vue()` 主要进行了初始化：合并配置、初始化生命周期、事件中心、render渲染、injection、data、provide、computed、watcher等；
  本章节，我们先研究主线任务，即 `模版和数据是怎样渲染成Dom的？` 而上面的初始化过程先跳过。

  - 在初始化的最后，Vue检测如果有 `el` 属性，则调用 `vm.$mount` 将它挂载到页面，挂载的目的也就是为了把模版渲染为最后的Dom，接下来继续分析挂载过程；


  ### Part3、个人实现的Vue仓库， 实现了模版编译、响应式等核心功能：
  [仓库地址](https://github.com/wlin00/simple-vue)


  


  



  

