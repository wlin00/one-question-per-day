instanceof方法，可以判断构造函数的prototype属性是否在实例对象的原型链上。
```javascript
  const my_instanceof = (a, b) => {
      a = a.__proto__
      b = b.prototype
      while (1) {
          if (a === b) return true
          if (a === null) return false
          a = a.__proto__
      }
  }
```