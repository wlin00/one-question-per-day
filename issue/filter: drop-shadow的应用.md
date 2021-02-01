当我们给一个div设置绝对定位的伪元素后，会发现div上的box-shadow不会成功设置给伪元素，如图：
![image](https://user-images.githubusercontent.com/48883217/105805581-3b2abd80-5fdd-11eb-81b6-200df0567ab2.png)

这时，可以使用filter: drop-shadow解决，效果如下：

```css
  filter: drop-shadow(0 0 2px #000);
```
![image](https://user-images.githubusercontent.com/48883217/105805387-da02ea00-5fdc-11eb-9748-a316208d8670.png)

filter:drop-shadow的兼容性相对box-shadow较差，仅在chrome 21+、Safari 6、ios 6下的Safari等浏览器支持。