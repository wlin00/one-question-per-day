  **场景1，异步线程池asyncPool**：
  实现一个 `asyncPool` 来实现promise的并发控制
  若当前有8个待办任务，但同一时间内只能最多有3个promise能够执行，例如大文件上传的场景
  当前3个执行队列的promise中 `最快的那个` 执行好后，程序会从 `待办任务列表` 获取到新的待办任务来添加到 `执行队列`

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

  **题目2，实现一个并发控制的调度器，保证同时最多运行2个任务**：
  当前最大长度为2的执行队列promise中 `最快的那个` 执行好后，程序会从 `待办任务列表` 获取到新的待办任务来添加到 `执行队列`
  示例
  ```typescript
    class Scheduler {
      constructor(limit = 2) {
        this._max = limit // 当前调度器最大并发数
      }
      addTask(task) { // task为传入的异步任务，是一个promise
        // 补全方法
      }
    }
    // 测试执行
    const sleep = (delay) => new Promise((resolve) => {
      setTimeout(() => {
        resolve()
      }, delay)
    })
    // 调度器使用
    const scheduler = new Scheduler(2)
    const addFn = (delay, id) => {
      // addFn 调用了 调度器实例的addTask方法，来根据并发请求数量决定，是执行当前任务还是挂起（等待最大并发数量小于限制）
      scheduler
        .addTask(() => sleep(delay))
        .then(() => {
          console.log(id)
        })
        .catch((err) => {
          console.log('err', err)
        })
    }
    // test - 若不存在并发控制，则打印1，2，3，4；若存在并发控制，则应该输出id顺序为：2、4、1、3
    addFn(4000, 4)
    addFn(2000, 2)
    addFn(3000, 3)
    addFn(900, 1)
  ```

  **Scheduler类代码实现**
  ```typescript
    class Scheduler {
      constructor(limit = 2) {
        this.max = limit // 当前调度器最大并发数
        this.working = [] // 正在工作的异步任务数组
        this.notWorking = [] // 当前暂时挂起的异步任务队列
      }
      addTask(task) { // task为传入的异步任务，是一个promise
        // 由于addTask可以走其后续的then & catch，说明它返回一个promise
        return new Promise((resolve, reject) => {
          // 重写异步任务的resolve和reject，即当前异步任务完成，可以改变addTask返回的promise状态，让外部调用方走后续操作
          task.resolve = resolve
          task.reject = reject
          // 判断当前最大并发数
          if (this.max > this.working.length) { // 允许直接执行当前任务
            this.runTask(task)
          } else { // 当前task任务需要挂起
            this.notWorking.push(task)
          }
        })
      }
      // 运行异步任务的方法
      runTask(task) {
        this.working.push(task)
        task().then(() => { // 执行异步任务，成功后，清空当前运行数组中的该异步任务 & 外部promise走then回调
          task.resolve()
          this.working.splice(this.working.indexOf(task), 1)
          // 若当前挂起队列中还有任务，则取出队列首部的一个任务，递归调用来清空异步任务
          if (this.notWorking?.length) {
            this.runTask(this.notWorking.shift())
          }
        }, (err) => { // 调用外部promise 的reject方法来让外部promise进行reject
          task.reject(err)
        })
      }
    }
    // 测试调度器执行
    const sleep = (delay) => new Promise((resolve) => {
      setTimeout(() => {
        resolve()
      }, delay)
    })
    // 调度器使用
    const scheduler = new Scheduler(2)
    const addFn = (delay, id) => {
      // addFn 调用了 调度器实例的addTask方法，来根据并发请求数量决定，是执行当前任务还是挂起（等待最大并发数量小于限制）
      scheduler
        .addTask(() => sleep(delay))
        .then(() => {
          console.log(id)
        })
        .catch((err) => {
          console.log('err', err)
        })
    }
    // test - 若不存在并发控制，则打印1，2，3，4；若存在并发控制，则应该输出id顺序为：2、4、1、3
    addFn(4000, 4)
    addFn(2000, 2)
    addFn(3000, 3)
    addFn(900, 1)
  ```