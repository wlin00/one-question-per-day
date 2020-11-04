## 介绍
  1、indexOf：可返回数组中某个指定的元素位置，返回数值
> 接受两个参数：1、要查询的元素  2、查询开始的坐标位置start
```
  当start > 0，start >= 数组长度arr.length时，返回 -1
  当start < 0，start绝对值 >= 数组长度arr.length时，则当前开始坐标是0
  当start < 0，start绝对值 < 数组长度arr.length时，则当前开始坐标是arr.length + start
```

  2、includes：可判断数组中是否包含某个元素，返回布尔值，ES6方法

> 接受两个参数：1、要查询的元素  2、查询开始的坐标位置start


  ## 其他区别
indexOf不能识别数组中的NaN，而includes可以识别数组中的NaN
```javascript
  var arr = [NaN]
  arr.indexOf(NaN) // -1
  arr.includes(NaN) // true
```
  当数组中存在空值时，includes会将其识别为undefined，而indexOf不会
```javascript
  var arr = []
  arr[4] = 'a'
  arr.indexOf(undefined) // -1
  arr.includes(undefined) // true
```

**indexOf兼容性** 可兼容至IE9

![image](https://user-images.githubusercontent.com/48883217/97544617-d4f5c500-1a04-11eb-970c-b9f10c5685ab.png)


**includes兼容性** ES6方法，兼容性相对较差

![image](https://user-images.githubusercontent.com/48883217/97545436-fc00c680-1a05-11eb-9ba3-5248207e658a.png)


## indexOf实现
```javascript
  Array.prototype.my_indexOf = function(value, start = 0) {
    if (start > this.length) {
      return -1
    }
    // 获取起始坐标
    if (start < 0) {
      start = start + this.length <= 0 ? 0 : this.length + start
    }
    // 从起始坐标开始查找目标
    for (let i = start; i < this.length; i++) {
      if (this[i] === value) return i
    }
    return -1
  }
```

## includes实现
```javascript
  Array.prototype.my_includes = function(value, start = 0) {
    if (start > this.length) {
      return false
    }
    // 获取起始坐标
    if (start < 0) {
      start = start + this.length <= 0 ? 0 : this.length + start
    }
    // 当前判断值是否是NaN
    if (Number.isNaN(value)) {
      for (let i = start; i < this.length; i++) {
        if (Number.isNaN(this[i])) return true
      }
    } else {
      // 从起始坐标开始查找目标
      for (let i = start; i < this.length; i++) {
        if (this[i] === value) return true
      }
    }
    return false
  }
```

[参考链接](https://segmentfault.com/a/1190000022959425?utm_source=tag-newest)