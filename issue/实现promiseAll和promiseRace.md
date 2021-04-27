## 1、Promise.all的实现
promise.all方法：
当有一个promise出错时，则reject其错误；或者当所有promise都resolve后统一返回一个结果数组
```javascript
const selfPromiseAll = (promises) => {
  return new Promise((resolve, reject) => {
    const resArr = []
    let index = 0
    const deal = (idx, res) => {
      index++
      resArr[idx] = res
      if (index === promises.length) {
        resolve(resArr)
      }
    }
    for (let i = 0; i < promises.length; i++) {
      Promise.resolve(promises[i]).then((res) => {
        deal(i, res)
      }, (err) => {
        return reject(err)
      })
    }
  })
}

```
## 2、Promise.race
返回最快执行的结果，不论状态是resolve或reject
```javascript
const selfPromiseRace = (promises) => {
  return new Promise((resolve, reject) => {
    for (let i = 0; i < promises.length; i++) {
      Promise.resolve(promises[i]).then((value) => {
        return resolve(value)
      }, (err) => {
        return reject(err)
      })
    }
  })
}
```
