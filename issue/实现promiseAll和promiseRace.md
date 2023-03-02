## 1、Promise.all的实现
promise.all方法：
当有一个promise出错时，则reject其错误；或者当所有promise都resolve后统一返回一个结果数组
```javascript
  Promise.myAll = (arr) => {
    return new Promise((resolve, reject) => {
      const res = []
      let index = 0
      for (let i = 0; i < arr.length; i++) {
        Promise.resolve(arr[i]).then((item) => {
          index++
          res[i] = item
          if (index === arr.length) {
            resolve(res)
          }
        }, (err) => {
          reject(err)
        })
      }
    })
  }
```
## 2、Promise.race
返回最快执行的结果，不论状态是resolve或reject
```javascript
  Promise.myRace = (arr) => {
    return new Promise((resolve, reject) => {
      arr.forEach((promise) => {
        Promise.resolve(promise).then((item) => {
          resolve(item)
        }, (err) => {
          reject(err)
        })
      })
    })
  }
```

测试代码
```typescript
  const sleep = (delay) => new Promise((resolve) => {
    setTimeout(() => {
      resolve(delay)
    }, delay)
  })
  const failSleep = (delay) => new Promise((resolve, reject) => {
    setTimeout(() => {
      reject(delay)
    }, delay)
  })
  // Promise.myAll test
  const arr = Promise.myAll([sleep(3000), sleep(4000), sleep(1500)]).then((res) => console.log('res', res), (err) => console.log('err', err)) // res 1500
  // Promise.myRace test
  const arr = Promise.myRace([sleep(3000), sleep(4000), failSleep(2500)]).then((res) => console.log('res', res), (err) => console.log('err', err)) // err 2500
```
