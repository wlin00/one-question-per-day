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