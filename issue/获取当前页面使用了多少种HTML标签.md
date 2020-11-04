解析

document.all可获取当前页面内所有元素的一个集合，是一个类数组，需要转换为真实数组处理。

```javascript
[...new Set(Array.prototype.slice.call(document.all).map(e=>e.tagName))].length
```