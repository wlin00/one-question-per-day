**章节1 Vue和React的区别 & 未来发展趋势**：
## 1）简介
- React: 核心是声明式渲染、组件化和单向数据流。利用纯函数的思想；写法是JSX+inline style 或css in js；
- Vue: 核心是组件化和双向绑定，是渐进式的MVVM框架；写法是sfc的模版语法，上手更容易。

## 2）响应式原理不同
```typescript
React：
  通过setState() 来更新状态，状态改变后，组件也会更新

Vue：
  Vue2通过发布订阅结合Object.defineProperty数据挟持来实现响应式；每个Vue组件实例都有一个对应的watcher实例，在组件初次渲染的时候会记录组件依赖了哪些数据，当数据发生改变时，会触发setter方法，并通知所有依赖这个数据的watcher，去调用update方法来触发组件的compile渲染方法，来实现更新视图。
  Vue3通过proxy来实现对对象内更多属性的挟持，使用懒代理等优化数据挟持的性能等；
```

## 3） react hooks的推出带来的影响 & 痛点 & 未来趋势

> 随着react16出现的react hooks，基本取代了现有的class components的开发，开拓了函数组件（组件逻辑表达、逻辑复用）的新范式；

随之而来的，其他框架有借鉴react hooks的思想，而也启发了后来的vue-composition api、Svelte3、SolidJs等
 
但随着React hooks被广泛使用，它的一些开发体验问题被逐渐正视，由于React是通过把 `组件的代码` 每一次 `更新` 都重复的调用，来模拟一些 `行为`，那么这就导致了一些反直觉的现状，如下：
```typescript
  1、不可以在React hooks中使用条件/循环语句，因react hooks是在fiber节点上存储hooks的链表，而每一次useState等的调用都是让链表的返回相应节点数据并进行移动一位，这就导致 条件/循环 可能会获取到错误的fiber节点；
  2、如果hooks不传入依赖数组，则会产生Stale Closure的过期闭包问题，这会带来心智负担；
  3、useEffect的依赖需要手动声明；
  4、需要 useMemo/useCallback 进行手动优化，否则会有性能问题。
```

React团队做的改善开发体验的努力
```typescript
  1、useEvent RFC改善 useCallback的依赖数组问题；
  2、Dan Abramov 花大量时间改进hooks文档；
  3、黄玄在开发中的React Forget，目的是作为编译时的优化工具，来自动为运行时的代码声明依赖。
```

基于依赖追踪的范式开始重新得到重视（Vue-composition api、SolidJs、Starbeam）
```typescript
  1、不同于hooks中禁写 条件/循环 语句，Vue3的setup则返回了一个闭包，内部的变量将可持续地被访问，而不是重复调用组件的代码，从而可以在条件循环中使用
  2、一次调用，符合JS直觉，不存在重复调用未声明依赖的hook，会产生过期闭包等问题；
  3、不同于useEffect需要手动声明依赖以及使用复杂，Vue3的watchEffect将会自动追踪依赖；
  4、引用稳定，无需useCallback；Vue3的setup中声明的所有函数都是固定的引用（return了包含自由变量 + 函数的闭包，而不是重复调用函数），所以不需要useCallback、useMemo的优化手段。
  5、Vue3 的proxy实现数据挟持，可以拦截到深层级的数据，同时使用weakMap收集已代理的依赖，方便垃圾回收用完的对象，采用了懒代理；同时将虚拟Dom性能与模版大小解耦，与动态节点数量正相关，并且compile会主动检测静态节点将他们提升到render之外来提示渲染性能，然后是Vue3使用了ESM导出全局APi和内部组件对Tree Shaking更友好；经尤大统计，Vue3运行时代码是13.5kb，而Vue2是22.5kb。这表示Vue3具备优越的性能；
  6、即使是这样Vue团队仍然在研究进一步提升性能的项目：Vue Vapor Mode，它支持在解析template的时候走另一个Vapor模式，一个更类似Solid的输出，是更具备卓越性能的模版策略。
```

**章节2 React18新特性总结**：
## 1）核心特征：concurrent
```ts
  concurrent：渲染可中断，以前的React中的update是同步渲染，在渲染任务完成前不可中断。React主要依靠自身fiber结构的链表来实现此功能，即改变fiber指针的指向来实现中断、废弃的update丢弃、状态复用等机制。
```
## 2）新增功能：React18基于concurrent基础，实现了如Suspense、transitions、流式服务端渲染等功能。
```ts
  1、createRoot：曾经使用ReactDOM.render创建一个初次的渲染或更新，现在使用createRoot，目的是复用创建root元素这一步骤
    const root = createRoot(document.getElementById('root')) // 根元素创建root对象，方便复用
    root.render(jsx)
  2、自动批量处理 Automatic Batching
    // react18以前setState 并不会批量处理，React 会 render2次
    setTimeout(() => {
      setCount(c => c + 1)
      setFlag(f => !f)
    })
    // react18: 自动批量处理，只render一次
    setTimeout(() => {
      setCount(c => c + 1)
      setFlag(f => !f)
    })
  3、Suspense：等待目前UI加载，并且可以直接指定一个加载的界面，让它在用户等待的页面显示；理念上Suspense有点像catch，只不过Suspense捕获的不是异常，而是组件的“挂载中”状态suspending
    <Suspense fallback={<Spinner />}>
      <Comments />
    </Suspense>
  4、transition：React把更新分成urgent update紧急更新 和 transition update过渡更新，过渡更新是把UI从一个视图改向另一个视图，这种更新优先级较低；使用 const [isPending, startTransition] = useTransition(), 也可更好代替setTimeout；startTransition相比setTimeout不会延迟调度而是立即执行，startTransition 会给本次更新添加一个“transition”的标记，而这个标记会作为React内部更新的一个参考条件；
  5、useDeferredValue：延迟更新某个不那么重要的部分，类似防抖debounce
    // state & defferdText
    const [text, setText] = useState('hello')
    const deferredText = useDeferredValue(text)
    // urgent ui
    <input value={text} onChange={handleChange} />
    // transition ui
    <MySlowList text={deferredText} />
    
```

