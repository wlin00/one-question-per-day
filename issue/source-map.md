**介绍：**
```
本文介绍常见的几种source-map的作用
```

source-map：为了解决开发代码与实际打包后代码不一致时，帮助我们定位错误代码、debug到文件原位置的技术。

webpack默认配置：eval-cheap-module-source-map

```typescript

  1、eval-source-map：每个模块都用eval（）执行，并且source map转化为 DataUrl 后添加到eval()中,可模块化缓存sourcemap；
  初始化较慢，但会具备较快重构建速度；
  2、cheap-source-map：大部分情况下我们不关心错误代码的“列”的信息，所以我们使用cheap模式可以提高sourcemap生成的效率；
  3、module-source-map：若没有加module那拿到的是loader处理器的代码如es6，若添加module则是经过loader处理过的代码如es5；
  4、hidden-source-map：sourcemap中不带源代码，只给出精准的错误行信息，且原文件末尾不引用sourcemap文件；
  5、nosources-source-map：sourcemap中不带源代码，只给出精准的错误行信息，避免信息泄漏；
  6、inline-source-map：将sourcemap 内联到原文件的末尾成为注释；
```