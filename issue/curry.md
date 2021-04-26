> 函数柯里化：在数学和计算机科学中，柯里化是一种将使用多个参数的一个函数转换成一系列使用一个参数的函数的技术。

curry 的这种用途可以理解为：参数复用。本质上是降低通用性，提高适用性。

实现
```javascript
function curry(fn, ...arg) {
  return (...rest) => {
    if (fn.length > [...arg, ...rest].length) {
      return curry.call(this, fn, ...arg, ...rest)
    } else {
      return fn(...arg, ...rest)
    }
  }
} 
```