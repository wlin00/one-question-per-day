**题目1、Vue2 和 Vue3 的区别**：
> 双向绑定
- Vue2 现有限制：
```typescript
  1、用Object.defineProperty() 实现双向绑定，默认只能处理一层的对象，如果是对象嵌套比较深的情况，则需要进行递归的数据挟持（深度遍历），即在observe内处理数据的set方法内部再次递归observe方法，直到处理的值不是一个引用数据类型；
  2、无法检测到 新的属性 的添加/删除，所以vue2提供了$set、$delete的api来追加数据挟持
  3、由于Object.defineProperty() 对数组的处理会有较高性能损耗，所以源码里对数组进行特殊处理：
    在 Observer类的构造函数里 判断若当前传入的data是一个数组，则让他继承于“自定义数组原型”，从而改写数组的七个方法，其中三个可新增数组元素的操作如push、unshift和splice，则是获取到他们新增出的值来额外为它们追加get和set的数据挟持。
```  
- Vue3 新增特性：
```typescript
  1、新增缓存机制 - 使用weakMap进行懒代理：
    // weakMap 只保留对象的弱引用，让垃圾回收机制能回收去除弱引用的对象
    // 使用正向缓存 - 判断当前入参target是否已经被代理过，若是直接返回缓存
    let toProxy = new WeakMap();
    // 使用负向缓存 - 判断当前如果传入的就是observed，则从反向缓存中查到存在则不进行代理，避免重复代理
    let toRaw = new WeakMap(); 
  2、Vue3 使用proxy进行对象代理，可以拦截所有对象上的操作；Proxy可以默认拦截数组类型
```  

> 虚拟Dom
```typescript
  // Vue2
  1、Vue2 虽然能保证触发更新的组件最小化，但在单个组件内部依然需要遍历该组件的整个vdom树；
  2、传统vdom性能跟模版大小正相关，跟动态节点的数量无关
  3、即Vue2 的更新是以组件为颗粒度来更新的， 但是单个组件的部分变化，仍然需要遍历整个vdom树

  // Vue3
  1、将AST基于动态节点指令（if、for、slot）切割为嵌套的区块，每个区块用数组来保存。更新的时候就只用去更新某个区块的节点
  2、Vue3的编译器Compiler 会主动检测模版中的静态节点，将静态节点提升到render函数之外；这样可以避免每次渲染的时候都重新创建静态节点的对象。提高内存使用、减少垃圾回收频率。
  3、给元素一个追踪标记PatchFlag，无论层级嵌套多深，更新时可以直接遍历动态节点，而不是整个Dom树更新
  4、Vue3将虚拟Dom节点渲染的性能与模版的大小进行解耦，将性能与动态节点的数量正相关，这带来整体性能提升2-5倍
```

> TreeShaking 
```typescript
  // Vue2
  Vue2 无论使用或不使用某个功能，都需要将它完整的下载，如内部自带的watch、compute、mounted等

  // Vue3
  Vue3 将全局的API和内部的一些组件都通过ESM导出，在用户使用的时候则按需求import，这使得webpack这种打包工具在打包的时候，Tree-Shaking会更加友好。（尤大在2020年4月给出：Vue2的运行时代码大小22.5kb，Vue3是13.5kb）
```

> Composition-api
```typescript
  将功能集成在setup里，让代码中相同的功能点能更加集中，并支持按功能点进行自定义hook的抽离，提高代码复用性。
```
> 代码管理：Monorepo
```typescript
  1、Vue3 将所有的模块统一放在一个主干分支，一个package.json下有多个文件夹
  2、Vue3中将各个功能抽离成一个个单独的模块，进行独立的打包，又能相互引用
``` 



**题目2 为什么react的hook不能在条件、循环语句中使用，而基于Hook理念的组合式api却可以**：
```typescript
  1、react追求纯函数理念, useState等hooks本质上是声明语句，为了声明和调用的一致性，不可避免地要强调顺序。而条件和循环会破坏这种顺序一致性。react hook是在fiber节点上存储hooks链表，每执行一次useState，返回相对应的节点数据，这时链表节点就会后移动一位，这意味着在循环/条件语句中调用，可能执行就会有调用次数少于链表节点移动次数，造成获取错误的fiber节点。

  2、vue3为什么没有这样的限制呢？
  export const TestVue3 = defineComponnt ({
    props: ...,
    setup: () => {
      let a
      if (window.name === 'test') {
        a = ref(0)
      }
      const b = ref(0)
      return () => (
        <div>
          {a && a.value} // 若使用条件语句 判断a是否为undefined
          {b.value}
        </div>
      )
    }
  })

  这是因为vue3的setup不是一个常规函数，而是一个含有闭包（闭包 = 自由变量 + 函数）的函数，即当改变ref.value触发重渲染的时候，不会重新执行setup函数，而是由 a变量、b变量、return的函数组成了一个闭包作用域，内部的变量可以被持续访问。才不会有react中找不到对应变量的问题，也是因为react把每次render中的useState的顺序 0、1、2、3 的值当作了key。
```


**题目3 第一次进入被keep-alive缓存的组件，会执行那些生命周期？第2次和第n次呢？**：
首先说下 `Vue的生命周期` 系统自带：beforeCreate、created、beforeMount、mounted、beforeUpdate、updated、beforeDestroy、destroyed
然后如果是加入了keep-alive会多两个生命周期：activated、deactivated
```typescript
  1、第一次进入被keep-alive缓存的组件，执行：
    beforeCreate、created、beforeMount、mounted、activated
    （离开路由，缓存组件失活的时候，执行deactivated）
  2、后续进入keepalive的组件，执行：
    activated
    （离开路由，缓存组件失活的时候，执行deactivated）
```

**题目4 哪些生命周期开始可以访问this.$el和this.$data？**：
```typescript
  1、this.$el: 在mounted阶段挂载页面后，可以访问到this.$el
  2、this.$data: 在created阶段，执行了initMixin -> initState -> 完成对data的初始化和数据挟持后，可以拿到this.$data
```

**题目5 Vuex是在哪个生命周期注入vue的？**：
```typescript
  1、我们在安装Vuex的时候，是通过Vue.use()来载入
    import Vuex from 'vuex'
    Vue.use(Vuex)
  2、Vue.use则是执行了插件的install方法，在Vue的initGlobalAPI里则有一个initUse方法来安装插件
    那么Vue.use是做了什么呢？
    Vue.use就是去判断传入的入参即插件是否存在，是的话直接返回；
    如果不存在则检测传入的plugin是否具备install方法，如果有的话，就执行插件的install：plugin.install.apply(plugin, args)
  3、看Vuex的源码，向外暴露了供Vue.use使用的install方法，它内部执行了Vue.mixin({ beforeCreate: vuexInit })
    所以可以知道，在Vuex的源码里，调用了Vue.minin来将vuexinit混入到beforeCreate这个生命周期钩子，那么后续每个vm实例都会在beforeCreate的阶段初始化Vuex
  4、vuexInit做了什么呢？
    1）如果判断当前选项$options里有store，则将store赋给this.$sotre, 说明是根节点。即this.$store = this.options.store
    2) 如果当前选项没有store，则不是跟节点，那么去父节点找这个stroe，即this.$store = this.$options.parent.$store
```

**题目6 描述一下Vue初始化的过程，模版和数据是怎样渲染为视图的？**：
  - `new Vue()` 主要进行了初始化：合并配置、初始化生命周期、事件中心、render渲染、injection、data、provide、computed、watcher等；
  本章节，我们先研究主线任务，即 `模版和数据是怎样渲染成Dom的？` 而上面的初始化过程先跳过。
  - 在初始化的最后，Vue检测如果有 `el` 属性，则调用 `vm.$mount` 将它挂载到页面，挂载的目的也就是为了把模版渲染为最后的Dom，接下来继续分析挂载过程；
```typescript
  1、初始化生命周期，是在Vue的构造函数内执行了 initLifeCycle，然后按顺序调用了$options 中的8个生命周期（如果有keep-alive则是10个）
  2、初始化事件中心，在Vue构造函数内执行了 initEvents，然后根据模版上的属性如@click去找到对应$options里的methods，去为对应的Dom节点添加addEventListener
  3、初始化data，主要是进行了props、methods、data的初始化，其中对于data初始化细节如下：
    1）取出当前选项$options中的data，typeof 判断其数据类型，如果是函数则调用函数，取它的返回值；如果是对象则直接返回；
    2）在this._data上存储当前data对象
    3) 对data对象进行递归的数据挟持，为内部所有类型的属性(数组、对象， 都添加getter和setter，实现方法是调用observe方法创建Observer的实例，内部对数据进行递归的挟持（Object.defineProperty)处理。
      对data内部属性做递归数据挟持的细节实现：
        如果是数组：
          一、Observer类的构造函数中，Array.isArray判断了当前data类型，如果是数组，则用forEach 处理数组然后对内部元素调用observe方法;
          二、并且将这个数组data的__proto__指向我们自定义的一个数组原型对象完成原型链的继承；这个自定义的数组对象则是Object.create(普通数组原型)的一个纯净副本，我们在这个自定义对象上做了特殊处理，重写了数组7大方法；
          特别是push、unshift、splice在新增数组元素的时候，我们将新增的数组也截取下来调用vm的对数组的挟持方法来为新增的属性添加 getter 和 setter。
        如果是对象：
          我们Object.keys遍历对象，来挟持对象内部属性，然后如果是set方法，执行完后，需要对新的值再次递归observe做递归数据挟持
    4)Object.keys遍历data的属性，使用Object.defineProperty(vm, key, { ... })对data中的属性进行代理，将他们代理到this._data对象上；这样我们就能通过this.xxx 访问 this._data.xxx
    5）模版解析，在mounted 阶段前，我们调用compile方法来解析当前模版，进行初始化视图渲染，细节如下：
      若当前选项传入了el, 则我们通过document.querySelect(this.$options.el) 来获取到目标挂载的Dom节点，然后我们判断节点类型：
        一、如果nodeType === 3 则是文本节点，我们写一个正则: reg = /\{\{(.*?)\}\}/ 来匹配括号内的内容，然后去对应data中找到这个key，替换给当前文本节点的textContent即可，如下：
          el.childNodes.forEach((item, index) => {
            if (item.nodeType === 3) {
              const reg = /\{\{(.*?)\}\}/ // 匹配括号内的内容，对内容的匹配是一个分组，.*?代表匹配除换行符外的任意字符
              const text = item.textContent.replace(reg, (match, key) => {  // replace的回调参数1:整个字符串；参数2:正则匹配后的字符串
                key = key.trim() // 对括号内的内容去除多余空格，获取对应data内的key
                return this.$data[key]
              })
              item.textContent = text
            }
          })
          
        二、如果nodeType === 1 则是元素节点，我们判断这个元素节点childNodes.length > 0 就递归compile函数解析它，直到解析为文本节点重复上述处理流程;同时这里可以添加事件监听，如下：
          el.childNodes.forEach((item, index) => {
            if (item.nodeType === 1) {
              // 添加事件监听
              // 若当前解析到元素节点， 需检测节点属性是否含有 @click、@mouseenter等，如果有则添加事件监听
              if (item.hasAttribute('@click')) {
                const methodKey = item.getAttribute('@click')
                item.addEventListener('click', this.$options.methods[methodKey].bind(this))
              }
              // 如果是元素节点 & 且不是空节点，则递归调用compile方法
              item.childNodes.length > 0 && this.compile(item)   
            }
          })
```

**题目7 Vue2中，数据修改，如何驱动视图修改？**：
```typescript
  1、Vue2的响应式原理：发布订阅模式 + 数据挟持
  2、初始化的过程中，我们在Vue构造函数中新建一个map的数据结构用于存储 某个属性被哪些视图watcher所引用（作为了发布订阅模式的第三方）
    this.$watchEvent = {}
  3、我们知道每个组件会对应一个watcher实例，所以我们需要一个Watcher类，在编译文本节点的时候为每个组件新建watcher；以便于后续数据变化可以结合全局watcher-map去通知依赖这个属性的每个watcher来更新视图
    class Watcher {
      constructor(node, attr, vm, key) {
        this.node = node
        this.attr = attr
        this.vm = vm
        this.key = key
      }
      update() {
        this.node[this.attr] = this.vm[this.key] // 用vm中新的属性值去更新视图
      }
    }
  4、在编译模版的文本节点时，我们记录那些使用了data中数据的节点，去存储它们的watcher; 即在全局watcher-map中以 《属性 -> 组件watcher实例数组》 的键值对的形式存储：
    代码如下：
    if (item.nodeType === 3) {
      // 如果是文本节点， 正则匹配括号内的内容，当作key去data中寻找，如果有的话，则替换data[key]
        const reg = /\{\{(.*?)\}\}/ // 匹配括号内的内容，对内容的匹配是一个分组，.*?代表匹配除换行符外的任意字符
        const text = item.textContent.replace(reg, (match, key) => { // replace的回调参数1:整个字符串；参数2:正则匹配后的字符串
          key = key.trim() // 对括号内的内容去除多余空格，获取对应data内的key
          // 在初始化用属性去渲染文本节点时，首先给每个组件实例绑定一个watcher，我们记录当前属性被哪些watcher所依赖
          let watcher = new Watcher(item, 'textContent', this, key)
          if (this.hasOwnProperty(key)) { // 若当前属性合法， 则记录属性被当前watcher所依赖
            if (this.$watchEvent[key]) {
              this.$watchEvent[key].push(watcher)  // 用于数据修改后，setter可以取出全局wacher-map中的对应属性的watcher数组，去通知每个watcher更新视图
            } else { // 若当前全局wacher-map没有当前属性，则初始化map的对应值为一个数组
              this.$watchEvent[key] = []
              this.$watchEvent[key].push(watcher)
            }
          }
          return this.$data[key]
        })
        // 将替换后的文本，修改给文本节点的textContent
        item.textContent = text
      }
  5、当observe中挟持到的数据发生变化，即set方法执行时，我们去判断这个属性是否有订阅者，如果则通知依赖这个属性的每个watcher去调用update方法更新组件视图
    observe的set中：
    set(newValue){
      if (newValue === value) {
        return
      }
      observe(target, newValue) // 递归数据挟持
      value = newValue // 状态更新

      // 视图更新, 若当前属性有订阅者，则让订阅的watcher们执行update
      if (this.$watchEvent[key]) {
        this.$watchEvent[key].forEach((item, index) => {
          item.update()
        })
      }
    }
  通过上述操作，可实现vue2的响应式
```

**题目7 Vue中的diff算法，初版**：
```typescript
  vue2中：
  1、比较两个vnode节点是否相等：
    1）比较两vnode的key
    2）比较两vnode的tag
    3）是否都定义了data，data包含的一些信息是否相等
    4）是否是input标签，是的话比较type
  若节点不等 ，则直接替换旧的节点oldVnode

  2、若两个vnode比对相等，进入patchVnode方法，来暴力比对两个节点的更多信息
    1）patchVnode过程主要根据根据节点类型判断来比对，如是否是文本节点（是的话更新textContent即可）
    2）核心：如果非文本节点且都有子节点，那么对两个vnode的子节点进行updateChildren方法

  3、updateChildren，用于多叉树的比对：
    1）oldStart 和 newStart 比对（头头），若相等下标++
    2）oldEnd 和 newEnd 比对（尾尾），若相等则下标--
    3）oldStart 和 newEnd 比对（头尾），若相等指针中间靠拢，且真实dom的第一个节点移动到最后
    2）oldEnd 和 newStart 比对（尾头），若相等指针中间靠拢，且真实dom的最后一个节点移动到最前
    5）若两个vnode有key：旧节点根据key生成hash-table，然后用newStart去一一匹配，指针继续中间靠拢，匹配成功将匹配的节点移动到真实dom的最前面
       若两个vnode无key：则直接将新节点头节点插入真实dom

  // vue的diff过程中，一个列表根据key对数组/列表进行重用的时候，使用的算法是《编辑距离》，表示如何用最小的修改次数将一个节点修改成新节点
    1)若两个vnode树末尾节点相等，则比对他们的前一位dp[i - 1][j - 1]
    2)若两个vnode树末尾节点不等，则取三种策略（新增、删除、替换）的最小值+1 ， dp[i][j] = Math.min(dp[i][j - 1], dp[i - 1][j], dp[i - 1][j - 1]) + 1
  

  vue3中：      
  vue3对于无脑的patchVnode做了优化，它在编译时会给节点添加静态节点（纯html元素）的类型，在后续patchNode的时候，如果是静态节点则跳过更新
```
