## 1、什么是Xss攻击?
  **Cross Site Script Attack（XSS）：一种通过html注入篡改网页、插入脚本来控制用户浏览器的攻击** 

> 他们可分为存储型、反射型、dom型

**举例如下：**
```
  ①、存储型：代码存储在服务器端，如果服务器未过滤或者过滤不严，其他用户触发这些代码会导致发生cookie窃取等攻击；
  ②、反射型：让用户点击后触发Xss，一般出现在搜索页面，非法用户可能用恶意脚本的url来收集用户信息；
  ③、Dom型：利用Dom这个与平台、语言无关的接口，改变页面的展示，利用Dom的一些属性来修改浏览器的展示。
```

## 2、Xss攻击的防范
```
  1、创建白名单：将敏感字符如<script>、&、#、%等进行过滤；
  2、对输出进行编码：对动态输出到页面上的内容进行转义和编码；
  3、react中慎用dangerouslySetInnerHtml方法：因react dom 在渲染的时候会转移为string类型；
  4、url检测：href、src必须以http://开头，白名单中不能有10进制和16进制字符；
  5、cookie设置http-only：js脚本不能读取cookie，减少xss的cookie挟持。
  6、cookie设置Secure属性，让cookie只能在https下传输。
```

## 3、防范Xss的新特性 - Content Secure Policy （内容安全策略）
- CSP 是什么？CSP设置后，只会允许加载指定的脚本及样式，最大限度防止 `XSS` 攻击，是解决XSS的最优解。可以在服务端设置 `Content Security Policy` 响应头来控制。

```
  1、可以通过指定域名来限制外部脚本是否可用，如： Content-Security-Policy：script-src 'self' （'self'代表只加载当前域名
  2、如果网站必须加载内联脚本（inline script），则可以提供一个 nonce 代表白名单，白名单外的域名无法注入脚本进行攻击。
```
- 以下是github的关于CSP的响应头配置
```s
Content-Security-Policy: 
  default-src 'none';
  base-uri 'self';
  block-all-mixed-content;
  connect-src 'self' uploads.github.com www.githubstatus.com collector.githubapp.com api.github.com www.google-analytics.com github-cloud.s3.amazonaws.com github-production-repository-file-5c1aeb.s3.amazonaws.com github-production-upload-manifest-file-7fdce7.s3.amazonaws.com github-production-user-asset-6210df.s3.amazonaws.com cdn.optimizely.com logx.optimizely.com/v1/events wss://alive.github.com;
  font-src github.githubassets.com;
  form-action 'self' github.com gist.github.com;
  frame-ancestors 'none';
  frame-src render.githubusercontent.com;
  img-src 'self' data: github.githubassets.com identicons.github.com collector.githubapp.com github-cloud.s3.amazonaws.com *.githubusercontent.com;
  manifest-src 'self';
  media-src 'none';
  script-src github.githubassets.com;
  style-src 'unsafe-inline' github.githubassets.com;
  worker-src github.com/socket-worker.js gist.github.com/socket-worker.js
```

[参考链接 - 阮一峰](http://www.ruanyifeng.com/blog/2016/09/csp.html)

[参考链接 - W3C](https://www.w3.org/TR/CSP3/#directive-form-action)