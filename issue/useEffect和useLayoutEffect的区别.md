## 介绍 
 **useEffect**和**useLayoutEffect**是reactv16.8 版本引入的hooks里的副作用钩子。在过去的做法里，我们常把拉取远端数据的请求放在componentDidMount里，而hooks出现后，可以让函数组件模拟class组件，可以在如**componentDidMount**、**componentDidUpdate**和**componentWillUnmount**等生命周期执行具备**副作用**的代码。

### 副作用是啥？
> 副作用：函数做了和本身运算返回无关的事情

  例如在函数中修改了全局变量、ajax请求、修改了Dom、修改入参甚至console.log都是具备副作用的。

## 应用
  当我们需要在useEffect中操作Dom，来看一段代码
```Jsx
import React ,{ useEffect, useRef } from 'react';
import TweenMax from 'gsap'
import './Demo1.scss'

const Demo1 = () => {
  const REI = useRef(null)
  useEffect(() => {
    // 0s的时间内， 将ref元素x轴方向移动600px
    TweenMax.to(REI.current, 0, {x: 600})
  }, [])

  return (
    <div className="Demo1">
      <div ref={REI} className="item">item1</div>
    </div>
  )
}

export default Demo1

```

  当我们修改页面保存后，可以看见被设置0秒内向右移动的方块item，在页面上出现了一个短暂的闪动。

![image](https://upload-images.jianshu.io/upload_images/8641818-b98fb38e8977d661.gif)

  原因是：useEffect的回调会组件渲染到屏幕后，即render结束后，异步执行，并且不会阻塞浏览器渲染，在这个钩子里操作Dom会引起浏览器额外的回流、重绘。

  而当换成useLayoutEffect后，刷新页面，元素就不会在页面上闪动，这是因为其回调会在Dom改变后同步执行，阻塞浏览器渲染，通过useLayoutEffect可以获取到最新的Dom。

![image](https://user-images.githubusercontent.com/48883217/99361241-c22c2d00-28ec-11eb-8cbf-07d23d164460.png)

### 总结
  useEffect：回调会在浏览器渲染的时候异步执行，不阻塞浏览器渲染。

  useLayoutEffect：回调会在Dom改变后同步执行，阻塞渲染。

## 为什么在处理Dom的操作放在useLayoutEffect？
  因为useLayoutEffect的回调会在Dom改变后同步执行，由于Js的引擎线程和浏览器渲染线程互斥，所以useLayoutEffect会阻塞浏览器的渲染，此时还未发生回流、重绘并且可以获取到最新的Dom。在这里操作Dom，react会在虚拟Dom提交到真实Dom后，也就是commitWork调用结束后，将Dom的改动和react的改动通过一次的回流、重绘渲染到页面上。

[参考链接](https://www.jianshu.com/p/412c874c5add)
[参考链接](http://www.ruanyifeng.com/blog/2019/09/react-hooks.html)