## 1、Referer字段是什么?
Referer是**Http请求头**里的一部分，用于表示当前请求页面的来源页面，和Origin不同的是，它含了路径信息。
当浏览器向web服务器发送请求时， 服务器可由此知道该网页是从哪个页面链接过来的。
通过Referer字段，可以**统计一个站点的流量、判断网站来源、进行日志分析**等。

> 补充：Referer的正确英文是referrer，但是早期HTTP拼写错误，为了保持兼容性就没有修改。

**举例如下：**
网站www.test.com请求了api/login的接口，那这次http请求的请求头中的Referer就会是www.test.com, 并且在Chrome85前，默认的策略referer-policy会允许referer带上来源页面上的请求参数。
如：www.test.com?query=1, 那么请求参数会在Referer上携带。

```Javascript
referer: www.test.com?query=1
```
 
## 2、Chrome85对referer的修改
原本的默认规则是：允许referer附带源网页上的参数，默认策略是：**no-referrer-when-downgrade**。
在Chrome85后，默认规则被修改为**strict-origin-when-cross-origin**，如果域名端口协议不一致，那么请求头上将不会携带源页面上的query参数。

## 3、 控制浏览器的referer策略
可以在前端页面上添加meta标签，来控制浏览器是否发送referer的相关策略, 即控制Referrer-Policy。
```html
<meta name="referrer" content="no-referrer-when-downgrade" />
<meta name="referrer" content="strict-origin-when-cross-origin" />
```

**referer的各个策略如下：**
**no-referrer**：请求头无referer字段
**origin**：只发送origin信息
**unsafe-url**：不论请求是否跨域，都具备referer字段，发送完整参数信息。
**strict-origin**: 协议降级的时候不发送referer
**no-referrer-when-downgrade**：协议降级的时候，无referer字段。
**origin-when-cross-origin**：跨域请求，referer字段发送origin信息，否则发送完整信息。
**same-origin**：跨域时没有referer字段，否则发送完整url信息。
**strict-origin-when-cross-origin**：同源时，发送完整url信息。协议降级的时候不会发送此首部。

可见各策略，在跨域、非跨域和协议是否降级的情况下具体的Referer差异。
![image](https://user-images.githubusercontent.com/48883217/96117298-76b4e680-0f1c-11eb-823c-c88afba02700.png)

[参考链接](https://www.yuque.com/alibabaf2e/gt3np7/nbcb65)
[参考链接](https://www.jianshu.com/p/26512475501a)