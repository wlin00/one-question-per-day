**1，防抖和节流**：

**代码：**

```JavaScript
  // 节流
  function throttle(fn, delay) {
    let timer
    return function (...arg) {
      let context = this
      if (!timer) { // 有定时器就不执行
        timer = setTimeout(() => {
          timer = null // 节流函数执行完需清空定时器
          fn && fn.apply(context, arg)
        }, delay)
      }
    }
  }

  // 防抖
  function debounce(fn, delay) {
    let timer
    return function (...arg) {
      let context = this
      if (timer) { // 每次定时器就清除
        clearTimeout(timer)
      }
      timer = setTimeout(() => {
        fn && fn.apply(context, arg)
      }, delay)
    }
  }
```