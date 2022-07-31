```typescript
function asyncPoll(tasks, maxProcess) {
  let i = 0;
  const ret = [];
  const excute = [];

  function enqueue() {
    if (i === tasks.length) {
      return Promise.resolve();
    }

    const p = tasks[i++]().then((res) => ({ status: 'fulfilled', input: res })).catch((e) => ({ status: 'rejected', input: e }));
    ret.push(p);
    const e = p.finally(() => excute.splice(excute.indexOf(e), 1));
    excute.push(e);
    let r = Promise.resolve();
    if (excute.length >= maxProcess) {
      r = Promise.race(excute);
    }
    return r.then(() => enqueue());
  }
  return enqueue().then(() => Promise.all(ret));
}

function createSuccess(input, time) {
  return () => new Promise((resolve) => {
    setTimeout(() => {
      resolve(input);
    }, time);
  });
}

function createError(input, time) {
  return () => new Promise((resolve, reject) => {
    setTimeout(() => {
      reject(input);
    }, time);
  });
}

asyncPoll([
  createSuccess(1, 1000),
  createError(2, 800),
  createSuccess(3, 1000),
  createError(4, 800),
  createSuccess(5, 1000),
], 2).then((res) => {
  console.log(res);
});
```