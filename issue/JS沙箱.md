## 需要JS沙箱的场景
```
1、多个微前端应用中，解决变量冲突的问题
2、当必须执行第三方js，且这个js不一定可信，则需要js沙箱提供一个独立的上下文作用域来保证其他运行环境不被污染和影响。
```
## JS沙箱需要实现的能力
```
1、创建一个独立的上下文作用域，其中的代码执行不会影响其他作用域环境
2、当有多个沙箱存在，每个都要具备加载、卸载、再次恢复等能力，对应着微应用的生命周期
```
## JS沙箱的实现核心点：
```
  1、核心是使用proxy对象创建window代理，并将需要被隔离起来的代码执行作用域绑定到proxy对象上
  2、属性active用来在外界控制沙箱是否运行
  3、new set() injectedKeys 用于记录添加的属性，在沙箱停止的时候删除
  4、使用Reflect 高级地实现属性操作
  5、with 和 call 把window的执行绑定到proxyWindow上
```
在微前端里
```
  1、基座初始化的时候，new sandbox
  2、ajax获取script标签的代码，并调用bindScope方法用eval在全局作用域执行
  3、webpack打包后动态加载的js使用 document.createElement 在head里增加了script标签，浏览器会动态执行。
  参考这个实现，可以使用document.createElement 复制并代理src属性的get和set，在set中获取js内容然后在沙箱执行。
```
## JS沙箱的实现：
```typescript
  export default class SandBox {
    constructor (name) {
      this.name = name
      this.active = false // 沙箱是否在运行，默认开启就运行
      this.micro_Window = {} // proxy代理到的目标对象
      const injectedKeys = new Set() // 新添加的属性，在卸载时清空
      this.proxyWindow = new Proxy(this.micro_Window, { //通过代理挟持window，当子应用修改或是用window上的属性时，把对应的操作记录下来在每次子应用挂载、卸载时生成快照，当再次从外部切到子应用时，又从记录的快照中恢复沙箱；又给予proxy代理所有全局性的常量和方法接口，为每个子应用构造了独立的运行环境。
        // 取值
        get: (target, key) => {},
        // 设置变量
        set: (target, key, value) => {},
        deleteProperty: (target, key) => {}
      })
    }
    start () { // 启动沙箱
      if (!this.active) {
        this.active = true
      }
    }
    stop () { // 停止沙箱 - 需清空变量
      if (this.active) {
        this.active = false
        // 把代理对象中目前记录的key清除掉
        this.injectedKeys.forEach((key) => { // Reflect 是Es6提供的一个静态类（不能new 去创建实例，只能调用这个静态类中的方法）
          Reflect.deleteProperty(this.micro_Window, key)
        })
        this.injectedKeys.clear() // 清空set
      }
    }
    bindScope (code) { // 修改js 作用域
      window.proxyWindow = this.proxyWindow
      return `;(function(window, self){with(window){;${code}\n}}).call(window.proxyWindow, window.proxyWindow, window,proxyWindow)`
    }
  }
```
