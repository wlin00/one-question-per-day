promiseArr实现
```ts
  Promise.myAll = function(arr) {
    return new Promise((resolve, reject) => {
      let index = 0, res = []
      for (let i = 0; i < arr.length; i++) {
        Promise.resolve(arr[i]).then((val) => {
          res[i] = val
          index++
          if (index === arr.length) {
            return resolve(res)
          }
        }, (err) => {
          return reject(err)
        })
      }
    })
  }
  // 测试代码
  const sleep = (i) => new Promise((resolve) => setTimeout(() => resolve(i), i))
  const fail = (i) => new Promise((resolve, reject) => setTimeout(() => reject(i), i))
  // 测试执行
  Promise.myAll([sleep(1000), fail(2000), sleep(3500)]).then((res) => console.log('res', res)).catch((err) => console.log('err', err))
```

promiseArr实现
```javascript
  Promise.myRace = function(arr) {
    return new Promise((resolve, reject) => {
      arr.forEach((item) => {
        Promise.resolve(item).then((val) => {
          return resolve(val)
        }, (err) => {
          return reject(err)
        })
      })
    })
  }
  // 测试代码
  const sleep = (i) => new Promise((resolve) => setTimeout(() => resolve(i), i))
  // 测试执行
  Promise.myRace([sleep(1000), sleep(2000), sleep(3500)]).then((res) => console.log('res', res))
```