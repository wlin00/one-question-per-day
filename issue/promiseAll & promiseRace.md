promiseArr实现
```javascript
const promiseAll = (promises) => {
  return new Promise((resolve, reject) => {
    let index = 0, resArr = [] 

    const deal = (idx, val) => {
      index++
      resArr[idx] = val
      if (index === promises.length) {
        return resolve(resArr)
      }
    }

    for (let i = 0; i < promises.length; i++) {
      Promise.resolve(promises[i]).then((val) => {
        deal(i, val)
      }, (err) => {
        return reject(err)
      })
    }
  })
}
```

promiseArr实现
```javascript
const promiseRace = (promises) => {
  return new Promise((resolve, reject) => {
    for(let i = 0; i < promises.length; i++) {
      Promise.resolve(promises[i]).then((value) => {
        return resove(value)
      }, (err) => {
        return reject(err)
      })
    }
  })
}
```