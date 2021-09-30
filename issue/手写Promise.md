**介绍：**
  Promise是是异步编程的一种解决方案，相比传统的解决方案回调函数和事件，Promise具备更精简易懂的语法。使用上，Promise通过 **then** 方法的链式调用，使得多层的回调嵌套看起来变成了同一层。

下面是回调函数和Promise的使用示例：
```javascript
// 回调函数
http.get('url',function(res) {
  console.log(res)
})

// Promise
new Promise(function(resolve){
  http.get('url',function(res){
    resolve(res)
  })
}).then(function(item){
  console.log(item)
})
```

上面的例子，看起来会感觉回调函数方法更加简洁，但当回调嵌套过多，会看到如下变化
```javascript
// 回调函数
http.get('url1', function (param1) {
    //do something
    http.get('url2', function (param2) {
        //do something
        http.get('url3', function (param3) {
            //dong something
            http.get('url4', function (param4) {
                //do something
            })
        })
    })
});
```

而Promise的写法则更加直观简洁
```javascript
//封装Promise
function getUserId(url) {
    return new Promise(function (resolve) {
        //异步请求
        http.get(url, function (id) {
            resolve(id)
        })
    })
}
getUserId('some_url').then(function (id) {
    //do something
    return getNameById(id); // getNameById 是和 getUserId 一样的Promise封装。下同
}).then(function (name) {
    //do something
    return getCourseByName(name);
}).then(function (course) {
    //do something
    return getCourseDetailByCourse(course);
}).then(function (courseDetail) {
    //do something
})
```

下面实现一个极简版的Promise：
```javascript
class selfPromise{
  callbacks = []
  constructor(fn) {
    fn(this._resolve.bind(this))
  }
  _resolve(value) {
    this.callbacks.forEach((callback) => callback(value))
  }
  then(onFulfilled) {
    this.callbacks.push(onFulfilled)
  }
}

// 使用
new selfPromise((resolve) => {
  setTimeout(() => {
    resolve('done')
  }, 5000)
}).then((res) => {
  console.log('res', res)
})
```
上述代码中，主要思路是：1、先使用then方法，收集所有想要在异步操作成功后的函数，将其放入callbacks队列，实际上是注册回调函数，类似观察者模式；
2、当异步操作成功后，会执行resolve方法，resolve会将callback异步函数队列中的方法取出来执行。

----------------------------------------------------------------------------------------------------------------------------------------------------------------
以上代码只实现了最基本功能，但这种方法没实现延时调用、链式调用、状态区分等大部分功能，这些功能的完成和思路将会后续在该文章中完善补充，下面我先贴出完整代码和核心思路。
  
**核心：链式调用的实现思路：**
  上一个Promise将控制权resolve给到return的新promise，让它将其当作onFulfilled方法使用。当下一个Promise状态改变后，调用此回调，则上一个Promise会拿到更新后的值并resolve，继续遍历callbacks。从而上一个Promise成功的获取到了下个Promise状态改变后的值而不是一个Promise对象，以此实现链式调用。

## 完整实现
```javascript
class selfPromise {
  callbacks = []
  state = 'PENDING'
  value = null
  constructor(fn) {
    fn(this._resolve.bind(this), this._reject.bind(this))
  }
  _reject(err) {
    this.value = err
    this.state = 'REJECT'
    this.callbacks.forEach((callback) => this._handle(callback))
  }
  _resolve(val) {
    // 判断当前传入的参数是不是一个promise， 若是则处理需要链式调用的场景
    if (val && ['function', 'object'].indexOf(typeof val) >= 0) {
      const then = val.then
      if (typeof then === 'function') {
        // 处理返回的Promise， 调用then方法，将上一个Promise的‘控制权’给到下一个
        then.call(val, this._resolve.bind(this), this._reject.bind(this))
        return
      }
    }
    this.value = val
    this.state = 'RESOLVE'
    this.callbacks.forEach((callback) => this._handle(callback))
  }
  then(onFulfilled, onRejected) {
    return new selfPromise((resolve, reject) => {
      this._handle({
        resolve: resolve,
        reject: reject,
        onFulfilled: onFulfilled || null,
        onRejected: onRejected || null,
      })
    })
  }
  catch(err) {
    this.then(null, err) // 重写catch方法
  }
  _handle(callback) {
    // 若当前状态未改变，则只注册事件（观察者模式）
    if (this.state === 'PENDING') {
      this.callbacks.push(callback)
      return
    }
    const fn = this.state === 'RESOLVE' ? callback.onFulfilled : callback.onRejected
    const run = this.state === 'RESOLVE' ? callback.resolve : callback.reject
    // 若当前未传入回调用方法, 则直接处理state中的状态
    if (!fn) {
      run(this.value)
      return
    }
    // 若传入回调，处理回调
    let res
    try {
      // 若此处是由return的新promise调用，则fn是等于调用了上一个promise的reolve
      // 调用后，上一个promise拿到最新的值，继续去遍历其callbacks，以此实现链式调用。
      res = fn(this.value) 
    } catch(error) {
      res = error
    } finally {
      run(res)
    }
  }
}
```
## 2、使用
```javascript
const promise11 = new selfPromise((resolve, reject) => {
  setTimeout(() => {
    resolve('resolve1')
  }, 1000)
})

const promise22 =  new selfPromise((resolve, reject) => {
  setTimeout(() => {
    resolve('resolve2')
  }, 3000)
})

promise11.then((res1) => {
  console.log('then1:',res1) 
  return promise22
}).then((res2) => {
  console.log('then2', res2)
}).catch((err) => {
  console.log('err', err)
})

// 输出结果： then1:resolve1   then2:reolve2
```
