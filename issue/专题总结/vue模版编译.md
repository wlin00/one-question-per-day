## vue生命周期各阶段示意图
![Image text](https://img-blog.csdnimg.cn/9e07fdc3b167468da4136fcb87ecb54d.png)

## vue2
可见，vue第一次渲染做了如下事情：
```typescript
  1、先执行initMixin中的 _init() 方法，来进行选项合并、生命周期、事件中心和状态等初始化；
  2、进入 created 阶段，则开始进行模版编译（若是runtime with compile版本则进行本地编译，若是runtime only版本则借助webpack和vue-loader进行离线编译，而两种编译都是为了得到render函数，方便下一步再由render函数生成虚拟Dom）
  3、模版编译先看有无 el， 或者是否有调用 $mount(el)，有的话走下一步
  4、先看当前vm.$options选项中是否有render function， 
    ->若没有:
      若选项中有template，则编译 template -> ast抽象语法树 -> render function；
      若选项中没有template，则获取当前el所在dom元素的outerHTML，编译 html -> ast抽象语法树 -> render 字符串 -> new Function来获取到render函数
    ->若有render，拿到render函数走mountComponent方法去进行挂载；实际上是先：render转虚拟Dom -> 虚拟Dom转真实Dom -> 真实Dom挂载到页面
      实际上是执行了：vm._update(vm._render())
  5、生成虚拟节点 VNode
  6、将虚拟Dom 转化为 真实Dom
  7、进行页面渲染    
```