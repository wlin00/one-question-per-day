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

  3、updateChildren，用于多叉树的比对：（由前两步骤保证了，当前的新旧vnode一定是同层级同类型的节点）
    核心：
    - 多叉树比对，在vue2叫：双端交叉diff
      若新旧vnode的头头、尾尾、头尾、尾头进行四次比对寻找到相同key的节点，则会进行复用，移动元素的位置；
      若四种情况都没有匹配到，但设置了key，则旧节点根据key生成hash-table，然后用newStart去一一匹配，看老的vnode里是否有当前节点，进行对应的移动、删除和创建
    - 多叉树比对，在vue3叫：双端快速diff
      第一步和vue3类似，进行四次双端比对
      第二步，若四次比对没有匹配上，则对新的vnode进行《最长递增子序列》算法，时间复杂度O（nlogn），
        该算法以原序列中的位置为基准，找到新vode的最长递增子序列列表，然后确定要这些子序列所对应的节点即需要保存的节点，然后将不在子序列中的节点移除，将需要添加的节点插入到指定位置，从而优化节点更新过程，减少DOM操作的次数，提高页面渲染性能。

    示例：
    1）oldStart 和 newStart 比对（头头），若相等下标++
    2）oldEnd 和 newEnd 比对（尾尾），若相等则下标--
    3）oldStart 和 newEnd 比对（头尾），若相等指针中间靠拢，且真实dom的第一个节点移动到最后
    2）oldEnd 和 newStart 比对（尾头），若相等指针中间靠拢，且真实dom的最后一个节点移动到最前
    5）若两个vnode有key：旧节点根据key生成hash-table，然后用newStart去一一匹配，指针继续中间靠拢，匹配成功将匹配的节点移动到真实dom的最前面
       若两个vnode无key：则直接将新节点头节点插入真实dom

  // vue2的diff过程中，一个列表根据key对数组/列表进行重用的时候，使用的算法是《编辑距离》，表示如何用最小的修改次数将一个节点修改成新节点
    1)若两个vnode树末尾节点相等，则比对他们的前一位dp[i - 1][j - 1]
    2)若两个vnode树末尾节点不等，则取三种策略（新增、删除、替换）的最小值+1 ， dp[i][j] = Math.min(dp[i][j - 1], dp[i - 1][j], dp[i - 1][j - 1]) + 1
  

  vue3中：      
  vue3对于无脑的patchVnode做了优化，它在编译时会给节点添加静态节点（纯html元素）的类型，在后续patchNode的时候，如果是静态节点则跳过更新
```

**题目8**
> Vue响应式原理
**从 `初始化` 到 `更新视图`**
```typescript
  一、首先core/instance/index.js 内是vue的真面目，是向外export function Vue() {} 作为构造函数，其内部调用了initMixin里实现的 this._init()方法；
  二、挂在原型prototype上的 _init() 方法，其实是做了一系列初始化（选项合并、生命周期、事件中心），当进入beforeCreate阶段（created阶段前），执行了 initState(), 这个方法内部获取了当前 vm.$options 然后对 props、methods和data 分别进行了初始化；
  三、其中 initState() 里对data的初始化就是 initData() 方法，这个方法主要做了如下事情：
    1、取出 vm.$options.data，判断如果是函数就取其返回值，否则取它本身（因data一般写成一个函数，避免属性重复）
    2、将当前data存储在 vm._data 上
    3、调用 observe() 方法来对当前data进行递归的数据挟持
      -> 处理数组：数组则重写原型上的七大方法，额外的挟持数组新增的属性；
      -> 处理对象：Object.keys() 遍历每个对象的key来调用 defineReactive() 方法进行数据挟持
      重点 - defineReactive() 方法详解：
        (1) 介绍： 
          这个方法主要是先根据当前key来进行 const dep = new Dep 生成一个dep实例即订阅器；后使用Object.definePropert()来将为当前data内部每个属性做递归的数据挟持，即set的时候会递归 observe(newValue);
        (2) get & set： 
          dep可用于后续数据获取，触发 get()方法时，可调用 dep.depend()来收集用到当前属性的watcher数组，将它们加入dep 
          dep也可用后续数据改变，触发 set()方法时，对新的值做递归observe数据挟持 & 更新值后，可调用 dep.notify()去通知当前dep中的依赖当前key的所有watcher 去调用其 update()方法更新视图；
        (3) 异步队列：
          watcher调用 update()更新视图的方式是：执行queueWatcher() 在js同步线程中收集watcher，而在执行的这些watcher的时候包裹了一层nextTick的异步队列，源码中选取异步队列的策略优先级是Promise > mutationObserver > setImmediate > setTimeout(0)；即当异步回调执行的时候，可保证同步任务中的需要更新的watcher已经收集完毕，此时可以调用watcher.run() 调用_update函数进行视图更新；
        (4) 渲染vnode：  
          render函数执行后，本质上是想用vnode生成真实dom，若是初始化运行，则patch中直接生成真实dom；否则进行patch和 updateChildren()方法的多叉树比对（即头头、尾尾、头尾、尾尾、比对key),最后完成视图渲染；
    4、Object.defineProperty 来将this.xxx 代理到 this._data.xxx
  四、进入$mount时期，若是 Vue Runtime with Compiler 的版本，则会先进行模版编译，即将模版转ast再转为render函数和Vnode；然后会执行new Watcher() 新建和当前视图（Vnode）相绑定的watcher实例；然后调用_update 初始化Vnode的时候，会取用Vnode上面的属性从而触发数据挟持的get，这时就会把watcher推入dep的数组中建立dep和渲染watcher的绑定关系。
```

**题目9**
> Keep-alive原理
  简介： `keep-alive` 是一个抽象组件，自身不会渲染为一个Dom元素；使用keep-alive包裹动态组件的时候，会 `缓存` 不活动的组件实例，而不是销毁他们。它在`列表` -> `文章详情`的场景常被用到

  使用：
  ```html
    <keep-alive include="/a|b/" exclude="b"> <!-- include代表需要缓存的组件名即白名单，exclude代表不会缓存的黑名单，并且优先级exclude更高 -->
      <component :is="view" /> <!-- 需要缓存的组件 -->
    </keep-alive>
  ```

  原理:Vue3中使用了LRUCache作为keep-alive实现原理
  ```typescript
    1、获取keep-alive包裹的第一个子组件对象以及其组件名；
    2、根据缓存黑白名单进行匹配，决定是否缓存，若不缓存则返回组件实例；若缓存则进行下一步；
    3、根据组件ID和tag生成缓存key，并在缓存对象中查找是否有这个key
      若存在：取出key并重新设置key来更新其使用时间（类似LRU）；
      若没有：在this.cache存储当前组件实例并保存key的值，再检查当前缓存实例个数是否超过最大值，是的话（在下一个新实例创建之前）删除最近最久未使用的那个缓存实例（下标为0的key）；

    vue3中的keep-alive是用了LRU算法：
      class LRUCache {
        capacity: number
        map: Map<number, number>
        constructor(capacity: number) {
          this.capacity = capacity
          this.map = new Map()
        }
        get(key: number): number {
          if (map.has(key)) {
            const value = map.get(key)
            map.delete(key)
            map.set(value)
            return value
          } else {
            return -1
          }
        }
        set(key: number, value: number): void {
          if (map.has(key)) {
            map.delete(key)
          }
          map.set(key, value)
          if (map.size > this.capacity) {
            const deleteKey = this.map.keys().next().value
            this.map.delete(deleteKey)
          }
        }
      }
  ```

**题目10**
> Vue - 事件中心
```ts
  - Vue的事件中心也基于发布订阅模式，模版编译的parseHTML时，vue会对属性和事件进行收集；
  - vm实例会创建一个对象来保存所有要监听的事件：vm._events = {}, 自定义事件中心主要支持: this.$on、$emit、$off、$once方法
  - 子组件初始化时，会走选项合并+initEvents，然后如果当前 VNode 有事件，则调用 updateComponentListeners
  - 原生 DOM 事件其实最终调用的还是原生事件的 addEventListener 和 removeEventListener 方法绑定和移出事件。patch 过程中的创建阶段和更新阶段都会执行 updateDOMListeners 方法,每次都会遍历 on 去添加事件监听，遍历 oldOn 去移除事件监听
  - vue对于事件的处理则是根据发布订阅map中的事件名key去操作对应的事件回调数组，批量地添加和移除监听
```

**题目11**
> Vue computed的实现原理
在`Vue`中，`computed`是一种特殊的`响应式数据`，他会根据依赖的数据动态地计算一个新的值，并且只有当依赖的数据发生改变的时候，它才会重新计算计算属性的值，并将结果缓存起来。
```ts
  // 实现原理
  1、将computed的属性转化为一个Watcher实例，这个watcher实例会收集computed所依赖的响应式数据，并只有当响应式数据变化的时候重新计算computed。在收集依赖的过程中，computed通过Dep类来建立计算属性和响应式数据之间的关系。
  2、第二步是在收集完依赖之后，将computed的值缓存起来，这个缓存过程是通过watcher实现的。在Watcher实例中，有一个dirty属性，用来表示computed是否需要重新计算（dirty为true代表需要重新计算）然后在初始情况，computed的值是未被计算的，dirty为true，当计算属性第一次被访问的时候，Watcher实例会进行计算并缓存计算结果；在之后的计算中，当computed依赖的响应式数据发生了变化才回重新计算并把dirty重置为true。
```

**题目12**
Vue Router 和 VueX 源码分析
```ts
1、Vue Router：https://blog.csdn.net/weixin_50786692/article/details/120527997
2、VueX：https://blog.csdn.net/weixin_39843414/article/details/103640539
```
