# 1、什么是跨站请求伪造?
  **Cross Site Request Forgery：一种挟持合法用户在已登录的web应用上执行恶意操作的攻击。** 

**举例如下：**
  ①、当用户进入银行站点'www.bank.com'
  ②、用户登录，输入账号密码，http请求成功；服务器在响应头设置set-cookie字段，浏览器收到请求会将代表自己身份的sessionId存储在cookie中，这样用户再次请求会携带sessionId以便服务器鉴权。
  ③、银行提供一个接口用于转账如：'www.bank.com/out?money=转账金额&to=目标账户'。
  ④、用户未退出银行网站，在同一个浏览器中不小心打开恶意网站B,此网站存在一个<img>标签:  
 
```html
<img src ='www.bank.com/out?money=1&to=abc'>
```  
用户点击图片会发起一个http请求，同时携带上用户登录获取的cookie，让银行服务器把用户的1块钱转给了非法用户。

**CSRF除了通过用户点击图片、链接来触发，还可能借助恶意的表单：**

```html
<form action="www.bank.com/transfer" method="POST">
  <input type="hidden" name="token" value="123">
  ...
</form>
```
通过诱导用户点击，或是借助浏览器自动提交表单来实现攻击。

# 2、Csrf防御的传统方式
**核心：http请求时，同一个域下的cookie会携带在请求头里提交**
  ①、设置验证码，让用户和应用强制交互。
  ②、服务器检查http请求头的字段Referer,禁止来自第三方的请求。
  ③、使用token令牌，表单的提交一般带一个随机token，告诉服务器这是真实的请求

# 3、防范Csrf新特性：SameSite
**从Chrome51开始，为更好的防范CSRF和用户追踪（请求头可伪造等原因），Cookie新增SameSite属性。**
  ①、SameSite=Lax，Chrome计划将lax设置为默认值，只有同站请求时发送cookie；
  ②、SameSite=Strict，只有同url请求时发送cookie(严格的限制，例如用户跳转url，不会带上附
加登陆状态的cookie)；
  ③、SameSite=None，跨站请求、同站请求都发送cookie(即sameSite失效，存在跨站请求伪造攻
击可能)。

[参考链接](http://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html)