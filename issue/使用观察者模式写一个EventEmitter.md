基于观察者模式 完成 EventEmitter 模块，它是一个类，它的实例具有以下几个方法：on、emit、off：

```javascript
const emitter = new EventEmitter()
emitter.on('hi', sayHi)
emitter.on('hi', sayHi2)
emitter.emit('hi', 'ScriptOJ')
// => Hello ScriptOJ
// => Good night, ScriptOJ

emitter.off('hi', sayHi)
emitter.emit('hi', 'ScriptOJ')
// => Good night, ScriptOJ

const emitter2 = new EventEmitter()
emitter2.on('hi', (name, age) => {})
emitter2.emit('hi', 'Jerry', 12)
// => I am Jerry, and I am 12 years old
```

实现
```typescript
  class EventEmitter {
    constructor() {
      this.events = {}
    }
    on(eventName, fn) { // 订阅
      if (!this.events[eventName]) {
        this.events[eventName] = []
      }
      this.events[eventName].push(fn)
    }
    off(eventName, fn) { // 取消订阅
      if (this.events[eventName]) {
        if (fn) {
          this.events[eventName].splice(this.events[eventName].indexOf(fn), 1)
        } else {
          this.events[eventName] = []
        }
      }
    }
    emit(eventName, ...args) { // 发布
      if (this.events[eventName]) {
        this.events[eventName].forEach((fn) => fn.apply(this, args))
      }
    }
  }
```