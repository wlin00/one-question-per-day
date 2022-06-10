> 我们项目打包上线后，一般能看到 <link real="preload"/> 和 <link real="prefetch"/> 这样的标签，本文来总结下两者区别。

```javascript
  1、preload: 对于页面有必要的资源使用，浏览器会提前加载这类文件，需要执行的时候再执行；可以将加载和执行分离开，不阻塞渲染和document的onload事件；
  2、prefetch: 告诉浏览器下一个页面可能会用的资源；
  一般在vue ssr的页面中，路由对应的资源使用prefetch，在首页的资源使用preload。
```
