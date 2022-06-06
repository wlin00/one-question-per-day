### 前言
该文章，对webpack原理和源码进行分析

### 第一章 - AST、Babel和依赖

## babel原理
1、parse - 通过@babel/parse 把代码转换成AST抽象语法树。
2、转换后的AST通过traverse进行遍历和修改。
3、generate： 将修改后的AST生成新的代码。

### 示例1 - 将代码中的let改成var
``` typescript
import { parse } from '@babel/parse'
import generate from '@babel/generate'
import traverse from '@babel/traverse'

// 源代码， 也可以使用fs从文件里读取
const code = `let a = 1; let b = '123'`
// 第一步， 代码转换成ast抽象语法树
const ast = parse(code, { sourceType: 'module' })
// 第二步，遍历ast，将let修改为var
traverse(ast, {
  enter: item => {
    if (item.node.type === 'VariableDeclaration' && item.node.kind === 'let') {
      item.node.kind = 'var'
    } 
  }
})
// 将ast重新转化生成新的代码
const result = generate(ast, {}, code)
// 输出
console.log(result.code)
```

### 示例2 - 将es6代码改成es5
``` typescript
import { parse } from '@babel/parse'
import * as babel from '@babel/core'

// 源代码， 也可以使用fs从文件里读取
const code = `let a = 1; let b = '123'`
// 第一步， 代码转换成ast抽象语法树
const ast = parse(code, { sourceType: 'module' })
// 第二步，使用babel能力：transformFromAstSync() 转换es6为es5
const result = babel.transformFromAstSync(ast, code, {
  presets: ['@babel/preset-env']
})
console.log(result.code)
```

也可以从文件中读取，修改es6为es5后，写到新文件中
``` typescript
import { parse } from '@babel/parse'
import * as babel from '@babel/core'
import * as fs from 'fs'

// 源代码， 也可以使用fs从文件里读取
const code = fs.readFileSync('./test.js').toString()
// 第一步， 代码转换成ast抽象语法树
const ast = parse(code, { sourceType: 'module' })
// 第二步，使用babel能力：transformFromAstSync() 转换es6为es5
const result = babel.transformFromAstSync(ast, code, {
  presets: ['@babel/preset-env']
})
// 将结果写到文件中
fs.writeFileSync('./test.es5.js', result.code)
```

### 示例3 - 对入口文件（code）进行递归的依赖分析
``` typescript
import { parse } from '@babel/parse'
import traverse from '@babel/traverse'
import { resolve, relative, dirname } from 'path'
import { readFileSync } from 'fs'

// 设置根目录 -- 获取需要解析的文件夹 - 根目录绝对路径
// __dirname: 返回当前模块文件解析过后所在的文件夹(目录)的绝对路径。
const projectRoot = resolve(__dirname, 'project_2')
// 类型设置 - 设置存放依赖关系的map的类型
type DepRelation = { [key: string]: { deps: string[], code: string } }
// 初始化map
const depRelation: DepRelation = {}

// 获取项目路径方法
const getProjectPath = (path: string) => { // 接收一个绝对路径， 返回相对于项目目录的相对路径
  return relative(projectRoot, path).replace(/\\/g, '/')
}

// 递归依赖收集方法
const collect = (filePath: string) => {
  // 获取当前收集的文件的项目路径（文件相对于项目根目录的相对路径）
  const key = getProjectPath(filePath)
  if (Object.keys(depRelation).includes(key)) {
    console.warn(`duplicated dependency: ${key}`) // 监测到重复key 退出递归
    return
  }
  // 获取源代码
  const code = readFileSync(filePath).toString()
  // 生成ast & map中记录当前依赖关系
  const ast = parse(code, { sourceType: 'module' })
  depRelation[key] = { 
    deps: [],
    code,
   }
  // 遍历ast， 进行依赖收集，且对于文件内部依赖的文件做递归分析
  traverse(ast, {
    enter: path => {
      if (path.node.type ==== 'ImportDeclaration') {
        // path.node.source.value 一般是一个相对路径，所以我们需要先转绝对路径， 再转项目路径用于依赖deps的收集
        // dirname返回路径中代表文件夹的那部分
        const depAbsolutePath = resolve(dirname(filePath), path.node.source.value) 
        const depProjectPath = getProjectPath(depAbsolutePath)
        // 依赖收集 & 递归分析
        depRelation[key].deps.push(depProjectPath)
        collect(depAbsolutePath)
      }
    }
  })
}

// 执行 - 传入入口文件绝对路径
collect(resolve(projectRoot, 'index.js'))
console.log(depRelation)
```

### 理解4 - 对webpack原理的理解
webpack用于解决两个问题：1、支持现代浏览器运行包含import、export的代码（Es-module）, 即把关键字转换成对应功能的函数；2、把所有文件打包成一个js

```
现代浏览器可以通过<script type='module'>来支持import和export，而IE8～15则不支持，并且这种方式会根据引入依赖加载所有文件，所以用webpack来进行依赖分析、把关键字转换为普通代码，并打包为一个js
```

### 示例5 - 现在入口文件index.js依赖了a.js和b.js， 我们对比babel/core转换版本前后的代码（es5）
```javascript
// index.js , 依赖分析入口文件
// 合理的循环引用
import a from './a.js'
import b from './b.js'
console.log(a.getB())
console.log(b.getA())

// a.js
import b from './b.js'
const a = {
  value: 'a',
  getB: () => b.value + 'from a.js'
}
export default a

// b.js
import a from './a.js'
const b = {
  value: 'b',
  getA: () => a.value + 'from a.js'
}
export default b
```

现在经过babel/core转换后的代码
![Image text](https://s3.bmp.ovh/imgs/2022/02/161270b06d6e4887.png)

下面对转换后的代码进行理解
```javaScript
  疑惑1：声明一个对象变量exports，将一个key写为'_esModule', 对应value写为true,即是exports['_esModule'] = true，用于符合CommonJS书写规范和约定。

  疑惑2: 同理初始化 exports['_esModule'] = void 0 即undefined。

  细节1: 在a.js中引入b.js，原ES语法的import被babel转为了_interopRequireDefault(require('./b.js'));  
  interopRequireDefault方法解析：判断入参是否是一个Common规范的模块（即obj['_esModule'] = true), 若是直接返回，若不是则将入参包装到{ 'default': obj }返回，加default目的是：CommonJS 模块没有默认导出，加上方便兼容。

  细节2: 即是等价于exports['default'] = a 等价于export了a模块。
```


### 示例6 - 使用上述能力写一个简易webpack打包器
```typescript
// 该文件模拟webpack打包核心功能，
  // 1、支持包括IE的浏览器运行ES6的代码，转换为es5；
  // 2、把所有文件打包成一个js，可直接任意浏览器中运行；
  // 3、depRelation数据结构改为一个对象数组，第一项为入口文件，作为collect递归依赖分析方法的入参。
  // 4、写一个generateCode方法来生成一个可运行的单js文件
// 可使用 node -r ts-node/register 文件路径 来运行，

import { parse }  from "@babel/parser"
import traverse from "@babel/traverse"
import { readFileSync, writeFileSync } from 'fs'
import { relative, resolve, dirname } from 'path'
import * as babel from '@babel/core'

// 设置根目录
const projectRoot = resolve(__dirname, 'project_3') // 获取项目根目录的绝对路径

// 类型设置
type DepRelation = { key: string, deps: string[], code: string  }[]

// 初始化map
const depRelation:DepRelation = []

// 获取文件相对于根目录的相对路径
const getProjectPath = (path: string) => { // 接收一个绝对路径，转换为如：index.js的项目路径
  return relative(projectRoot, path).replace(/\\/g, '/')
}

// 依赖收集函数
const collect = (filePath: string) => {
  // 获取当前收集文件的项目路径如：index.js
  const key = getProjectPath(filePath)
  if (depRelation.find(item => item.key === key)) {
    console.warn(`duplicated dependency: ${key}`) // 监测到重复key 不进行依赖收集
    return
  }
  // 获取源代码 - 通过babel/core的能力将代码转换为兼容的es5版本 - import 转为 require， export转为exports['default'] = aaa
  const code = readFileSync(filePath).toString()
  const { code: es5Code } = babel.transform(code, {
    presets: ['@babel/preset-env']
  })
  // 生成ast， 并将当前依赖放入depRelation中
  const dependency = {
    key,
    deps: [],
    code: es5Code,
  }
  depRelation.push(dependency)
  const ast = parse(code, { sourceType: 'module' })

  // 遍历ast， 进行依赖分析&收集
  traverse(ast, {
    enter: path => {
      if (path.node.type === 'ImportDeclaration') {
        // path.node.source.value 往往是一个相对路径， 所以我们需要转绝对路径 - 再转项目路径，方便放入依赖map的deps
        const depAbsolutePath = resolve(dirname(filePath), path.node.source.value)
        const depProjctPath = getProjectPath(depAbsolutePath)
        // 依赖收集
        // depRelation[key].deps.push(depProjctPath)
        dependency.deps.push(depProjctPath)

        // 递归分析
        collect(depAbsolutePath)
      }
    }
  })

}

// 依赖收集
collect(resolve(projectRoot, 'index.js')) // 函数参数是：人口文件的绝对路径

console.log(depRelation)

// 收集完后， 根据产出的依赖对象数组，产出一个可独立运行js代码，可通过writeFileSync写入文件。
/** 最终文件主要内容
  var depRelation = [ 
    {key: 'index.js', deps: ['a.js', 'b.js'], code: function... },
    {key: 'a.js', deps: ['b.js'], code: function... },
    {key: 'b.js', deps: ['a.js'], code: function... }
  ] 
  var modules = {} // modules 用于缓存所有模块
  execute(depRelation[0].key) // 运行入口文件
  function execute(key){
    var require = ... // 引入模块即运行一次模块即可
    var module = ...
    item.code(require, module, module.exports)
    ...
  }
 * 
 */
const generateCode = () => {
  let code = ''
  code += 
    'var depRelation = ['
    + depRelation.map((item) => {
      const { key, deps, code } = item
      return `{
        key: ${JSON.stringify(key)},
        deps: ${JSON.stringify(deps)},
        code: function(require, module, exports){
          ${code}
        }
      }`
    }).join(',')
    + '];\n'
  code += `var modules = {};\n`
  code += `execute(depRelation[0].key)\n`
  code += `
    function execute(key) { // 找到传入key的依赖，运行内部代码
      if (modules[key]) { return modules[key] } // 若模块中有该key，直接返回
      var dependency = depRelation.find((item) => item.key === key)
      if (!dependency) {
        throw new Error(\`\${key} is not found\`)
      }
      var pathToKey = (path) => {
        var dirname = key.substring(0, key.lastIndexOf('/') + 1)
        var projectPath = (dirname + path).replace(\/\\.\\\/\/g, '').replace(\/\\\/\\\/\/, '/')
        return projectPath
      }
      var require = (path) => {
        return execute(pathToKey(path))
      }
      modules[key] = { __esModule: true } // 缓存当前模块
      var module = { exports: modules[key] } // module用于CommonJs的兼容和约定写法
      dependency.code(require, module, module.exports)
      return modules[key]
    }
  `
  return code
}

writeFileSync('dist.js', generateCode())

```

执行打包器后，将会生成一个可执行的、由es5组成的js文件dist.js, 可由node dist.js执行，运行打包入口文件的代码。


### 示例7 - 简易webpack打包器新增对css文件的解析能力，手写一个loader来支持

首先，假设index.js入口文件中多引入了一个css文件
```javascript
import a from './a.js'
import b from './b.js'
// 引入css
import './style.css'
console.log(a.getB())
console.log(b.getA())
```

css文件就是修改了字体颜色
```css
  body {
    color: red;
  }
```

接下来我们对打包器进行修改，让它在判断当前项目路径是css文件，对css进行解析转为js，改进后代码如下：
```typescript
// 该文件模拟webpack打包核心功能，
  // 1、支持包括IE的浏览器运行ES6的代码，转换为es5；
  // 2、把所有文件打包成一个js，可直接任意浏览器中运行；
  // 3、depRelation数据结构改为一个对象数组，第一项为入口文件，作为collect递归依赖分析方法的入参。
  // 4、写一个generateCode方法来生成一个可运行的单js文件
// 可使用 node -r ts-node/register 文件路径 来运行，

import { parse }  from "@babel/parser"
import traverse from "@babel/traverse"
import { readFileSync, writeFileSync } from 'fs'
import { relative, resolve, dirname } from 'path'
import * as babel from '@babel/core'

// 设置根目录
const projectRoot = resolve(__dirname, 'project_3') // 获取项目根目录的绝对路径

// 类型设置
type DepRelation = { key: string, deps: string[], code: string  }[]

// 初始化map
const depRelation:DepRelation = []

// 获取文件相对于根目录的相对路径
const getProjectPath = (path: string) => { // 接收一个绝对路径，转换为如：index.js的项目路径
  return relative(projectRoot, path).replace(/\\/g, '/')
}

// 依赖收集函数
const collect = (filePath: string) => {
  // 获取当前收集文件的项目路径如：index.js
  const key = getProjectPath(filePath)
  if (depRelation.find(item => item.key === key)) {
    console.warn(`duplicated dependency: ${key}`) // 监测到重复key 不进行依赖收集
    return
  }
  // 获取源代码 - 通过babel/core的能力将代码转换为兼容的es5版本 - import 转为 require， export转为exports['default'] = aaa
  let code = readFileSync(filePath).toString()

  // 注意， 这里开始判断是否是css --------------------------------------------------------------

  if (/.css$/.test(code)) {
    // 若是css， 则特殊处理代码，css转JSON字符串再插入到style标签
    code = require('./utils/style-loader.js')(code) // loader代码看下方
  }
  const { code: es5Code } = babel.transform(code, {
    presets: ['@babel/preset-env']
  })
  // 生成ast， 并将当前依赖放入depRelation中
  const dependency = {
    key,
    deps: [],
    code: es5Code,
  }
  depRelation.push(dependency)
  const ast = parse(code, { sourceType: 'module' })

  // 遍历ast， 进行依赖分析&收集
  traverse(ast, {
    enter: path => {
      if (path.node.type === 'ImportDeclaration') {
        // path.node.source.value 往往是一个相对路径， 所以我们需要转绝对路径 - 再转项目路径，方便放入依赖map的deps
        const depAbsolutePath = resolve(dirname(filePath), path.node.source.value)
        const depProjctPath = getProjectPath(depAbsolutePath)
        // 依赖收集
        // depRelation[key].deps.push(depProjctPath)
        dependency.deps.push(depProjctPath)

        // 递归分析
        collect(depAbsolutePath)
      }
    }
  })

}

// 依赖收集
collect(resolve(projectRoot, 'index.js')) // 函数参数是：人口文件的绝对路径

console.log(depRelation)

// 收集完后， 根据产出的依赖对象数组，产出一个可独立运行js代码，可通过writeFileSync写入文件。
/** 最终文件主要内容
  var depRelation = [ 
    {key: 'index.js', deps: ['a.js', 'b.js'], code: function... },
    {key: 'a.js', deps: ['b.js'], code: function... },
    {key: 'b.js', deps: ['a.js'], code: function... }
  ] 
  var modules = {} // modules 用于缓存所有模块
  execute(depRelation[0].key) // 运行入口文件
  function execute(key){
    var require = ... // 引入模块即运行一次模块即可
    var module = ...
    item.code(require, module, module.exports)
    ...
  }
 * 
 */
const generateCode = () => {
  let code = ''
  code += 
    'var depRelation = ['
    + depRelation.map((item) => {
      const { key, deps, code } = item
      return `{
        key: ${JSON.stringify(key)},
        deps: ${JSON.stringify(deps)},
        code: function(require, module, exports){
          ${code}
        }
      }`
    }).join(',')
    + '];\n'
  code += `var modules = {};\n`
  code += `execute(depRelation[0].key)\n`
  code += `
    function execute(key) { // 找到传入key的依赖，运行内部代码
      if (modules[key]) { return modules[key] } // 若模块中有该key，直接返回
      var dependency = depRelation.find((item) => item.key === key)
      if (!dependency) {
        throw new Error(\`\${key} is not found\`)
      }
      var pathToKey = (path) => {
        var dirname = key.substring(0, key.lastIndexOf('/') + 1)
        var projectPath = (dirname + path).replace(\/\\.\\\/\/g, '').replace(\/\\\/\\\/\/, '/')
        return projectPath
      }
      var require = (path) => {
        return execute(pathToKey(path))
      }
      modules[key] = { __esModule: true } // 缓存当前模块
      var module = { exports: modules[key] } // module用于CommonJs的兼容和约定写法
      dependency.code(require, module, module.exports)
      return modules[key]
    }
  `
  return code
}

writeFileSync('dist.js', generateCode())

```

处理css为js， 并创建style标签插入body的loader
```javascript
  // 采用es5和js，原因是兼容各版本浏览器
  const transform = (code) => {
    var res = `
      var str = ${JSON.stringify(code)}
      if (document) {
        var style = document.createElement(style)
        style.innerHTML = str
        document.head.appendChild(style)
      }
    `
    return res
  }
  module.exports = transform
```

写完这个loader，我们需要记住关于loader的一些事
```javaScript
  1、单一职责原则，可见上面我们的loader做了两件事，转css为js 以及 插入代码到style标签 这其实不符合单一职责原则。
  2、一个功能可能是由多个loader同时协同处理 ，例如sass-loader把scss文件转换为css；再由cssloader转css为js；再由style-loader创建style标签；
```

关于loader面试题
```javaScript
问：webpack的loader是什么？
答：
  1、webpack自带的打包器只能处理js文件；
  2、当我们想加载css/less/scss/ts/md等文件时，就需要特定的loader来提供特殊功能；
  3、laoder的原理是将文件转换成可处理的js（webpack由第三方库Acorn来进行JavaScriptParse）
  4、例如：css-loader将css代码转换变成了export default str形式，来用于后续的str插入style标签；
  5、深入了解源码需要学习：webpack的pitch钩子和request对象。
```

## webpack源码阅读
一、webpack-cli 是如何调用 webpack 的
```JavaScript
  1、在demo目录，运行node_modules/.bin/webpack-cli时，会借助webpack能力默认帮我们将入口/src/index.js打包到dist目录成为main.js。
  2、阅读webpack-cli源码，找到对应的主文件/lib/webpack-cli.js, 发现在控制台执行的webpack-cli的打包能力来自于webpack依赖里的bin目录下的cli.js
  3、阅读bin/cli.js的代码，发现其调用了lib/bootstrap.js引导程序的能力，来创建了一个WebpackCli的实例，并调用了原型上的run方法
  4、阅读lib/webpack-cli.js构造函数的代码，发现其原型上的run方法实际上是调用require了webpack的依赖来创建了一个解析器，即createCompiler。
    const webpack = packageExists('webpack') ? require('webpack') : undefined;
    compiler = webpack(options, callback);
  5、总结：控制台中使用命令webpack-cli xxx.js，实际上是webpack-cli是通过调用了bin目录下的cli.js -> lib/boostrap.js的能力来创建了一个WebpackCli对象的实例并调用其原型上的run方法来导入webpack能力创建了一个解析器，从而完成对入口文件的打包。
```

二、webpack整体阶段
```JavaScript
  1、从"lib/webpack.js"里的createCompiler方法开始，触发了第一个由tabable绑定的事件，下面记录核心的事件&函数的执行顺序来整理体现出webpack的工作流程
  2、顺序如下：
    // lib/webpack.js中的createCompiler方法 - 创建解析器（return了一个Compiler类的实例）, 内部触发三个事件。
    @environment
    @afterEnvironment
    @initialize
    // lib/webpack.js主方法，创建compiler后，继续执行webpack.js主方法上的compiler.run
    @beforeRun
    @run
    this.readRecords()
    this.compile(onCompiled)
    // Compiler.compile - 编译流程
    @beforeCompile // 预编译
    @compile // 编译事件
    this.newCompilation() // 生成一个新的编译
      // newCompilation - 内部
      this.createCompilation() // 初始化了一个“编译”
      @thisCompilation
      @compilation
    @make
    @finishMake    
    compilation.finish
      // compilation.finish - 内部
      @finishModules
    compilation.seal
      // compilation.seal - 内部 - 优化
      @seal
      @afterOptimizeDependencies
      @beforeChunks
      this.addChunk(name)
      this.buildChunkGraph()
      @afterChunks
      @optimize
      @afterOptimizeModules
      @afterOptimizeChunks
      @optimizeTree
    @afterCompile

    // onConpiled的回调
    @shouldEmit
    this.emitAssets()
    this.emitRecords()
    @done
```

三、webpack如何分析index.js的
```JavaScript
  1、首先分析webpack是向外导出了什么文件，来提供了的打包能力；为此我们先看webpack/package.json文件，找到其主要导出文件"main":"lib/index.js"；
  2、阅读"lib/index.js"源码，发现其核心向外导出了一个fn函数，这个函数的核心是require引入了相邻目录下的"./webpack.js";
  3、阅读"lib/webpack.js"源码，发现其向外暴露了一个箭头函数webpack，而该函数内部是调用了"createCompiler"方法来创建解析器，该方法实质上是创建了一个"Compiler"类的实例：
    const compiler = new Compiler(options.context);
  然后触发了基于webpack官方库《tabable》绑定的，在这个实例上的几个事件：
    compiler.hooks.environment.call();
	  compiler.hooks.afterEnvironment.call();
	  // new WebpackOptionsApply().process(options, compiler);
    compiler.hooks.initialize.call();
  4、通过以上操作持续分析webpack阅读源码，总结：
    webpack使用其官方发布订阅的库Tapable作为事件中心，将打包过程分为了几个重要的阶段如：env、compile、compilation、make、finishMake、seal、emit等；
    在make阶段，由入口插件（EntryPlugin.js）来完成对入口文件的解析 src/index.js；
    并且在这个阶段创建了一个Module的数据结构（一般是NMF，normal module factory）来存储当前打包的文件依赖相关的信息；
    在make阶段的buildModule事件里，webpack先通过执行runLoaders方法对非js文件进行处理，然后webpack借助第三方库Acorn处理js为AST（webpack的本职工作也是借助Acorn做JavaScriptParse），然后就可以对AST做traverse操作进行递归的依赖分析，一旦发现当前ast节点的type为‘ImportDeclaration’，就把这依赖的子文件添加到当前module的dependencies中；
    在后面的seal阶段，这个阶段的主要职能是对module进行封装，webpack将module转化为一个或多个chunk；
    seal之后，webpack为每个chunk创建文件（bundle），在emit阶段写文件到硬盘上。
```

四、webpack中，module、chunk、bundle的关系
[参考链接](https://www.jianshu.com/p/040323107958)

五、webpack的插件plugin介绍
```javaScript
  1、loader和plugin的关系
    loader：loader是文件加载器，能够加载资源文件，并对文件做压缩、编译等处理，最终将处理的文件打包到结果的bundle文件中。
    plugin：plugin是webpack的拓展器，在webpack运行的生命周期中会有多个阶段的事件（基于Tapable作为事件中心的发布订阅架构），而插件就是可以对webpack的每一个阶段做监听，可以插入到每一个打包阶段，来实现各式各样的功能，丰富webapck本身能力。
  2、插件1：imagemin-webpack-plugin 能力：每次文件打包写文件前，检测当前依赖是否是图片，若是的话对其优化压缩，返回新的优化图片。
    在 compiler.hooks.emit.tapAsync 的回调中执行插件能力（即监听emit阶段），遍历compilation.assets编译文件，若发现当前编译文件是图片则调用optimizeImage来优化图片。
  3、插件2: clean-webpack-plugin 能力：每次文件打包写文件前，清空output输出目录；打包结束后，在done的事件阶段删除所有没用到的文件如垃圾文件、临时文件。
    该插件会在compile.hooks.emit.tap 的回调中执行插件能力（即监听emit阶段），若本次编译没出错，则在写文件到硬盘之前，清空output输出目录的文件；
    在done阶段，compile.hooks.done.tap 的回调中，遍历assets文件目录，删除所有要用的文件之外的文件如临时文件、垃圾文件。
  4、插件3：ProvidePlugin 能力：帮用户全局引入某个依赖，让后续文件中不用再每次都引入。
    该插件会在compile.hooks.compilation.tap 的回调中执行插件能力（即监听compilation阶段），该插件在compilation阶段获取到nmf普通模块工厂，并继续监听其原型上的parse事件（监听代码转换为ast的parse阶段）；该插件会在代码借助Acorn库转化为ast的过程中执行能力 - 遍历的 definition 配置，然后为当前模块引入所有 expression 表达式里用过的但未引入的依赖；
    即一个全局注入引入依赖的能力，配置后就不用在每个文件import vue from 'vue'
```

六、写一个plugin的5个要点
```javascript
  1、向外导出一个JavaScript命名函数或一个类；
  2、在插件函数的原型 or 类的方法上定义一个apply，webpack默认作为该插件能力；
  3、监听一个webpack自身通过Tapable发布的事件（如compile.hooks.compilation.tap)；
  4、webpack内部实例的数据处理；
  5、功能完成后调用 webpack 提供的callback回调。
```

下面写一个插件，其功能是：生成一个叫做 assets.md 的新文件；文件内容是所有构建生成的文件的列表。
```typescript
  class FileListPlugin {
    // 默认文件名
    static defaultOptions = {
      outputFile: 'assets.md'
    }
    constructor(options = {}) {
      // 合并后的选项暴露给插件方法
      this.options = { ...defaultOptions, ...options }
    }
    apply(compiler) {
      const pluginName = FileListPlugin.name
      // 当前webpack模块的实例，可以从compiler中解构获取
      const { webpack } = compiler
      // 再解构webpack实例获取Compilation对象，里面装了一些有用的常量
      const { Compilation } = webpack
      // RawSource 是一种源码类型，可以用它创建一个文件，然后在写文件的阶段emitAsset，额外地添加一个文件写到output目录
      const { RawSource } = webpack.sources

      // 监听webpack的thisCompilation阶段, 这个阶段可以获取compilation编译
      compiler.hooks.thisCompilation.tap(pluginName, (compilation) => {
        compilation.hooks.processAssets.tap({
          name: pluginName,
          // 用较靠后的资源处理阶段，保证当前processAssets阶段所有资源被添加到 compilation
          stage: Compilation.PROCESS_ASSETS_STAGE_SUMMARIZE 
        }, (assets) => {
          // assets 是包含了当前编译中的所有资源的对象，key是资源路径 - value是源码
          // 下面需要遍历所有资源，获取回车分割的文件路径列表的md文件源代码，然后经过RawSource封装一起emitAsset写到文件中
          const content = '# In this build: \n\n' +
            Object.keys(assets).map((filename) => `- ${filename}`).join('\n')

          compilation.emitAsset(
            this.options.outputFile,
            new RawSource(content)
          )  
        })
      })
    }
  }

  module.exports = { FileListPlugin }
```


七、webpack高级配置
```javascript
  1、让webpack5支持ie浏览器（因webpack5开始不默认支持ie）,做法是在.browserslistrc文件里配置兼容浏览器
  代码：
    [production] // 开发环境
    > 1% // 支持全世界 > 1%的浏览器 + ie9
    ie 9

    [modern]
    last 1 chrome version // 支持最新的1个谷歌和火狐浏览器
    last 1 firefox version

    [ssr]
    node 12

  2、在webpack5中使用babel-loader来打包js文件
  module.exports = {
    mode: 'production',
    module: {
      rules: [
        {
          // 用babel-loader来处理js/jsx文件，而非用webpack默认能力打包
          // 这样方便拓展更多能力
          test: /\.js$/,
          exclude: /node_modules/, // 遇到node_modules文件不处理，因为这些文件都默认打包过
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                ['@babel/preset-env'] // 使用babel-loader预设能力处理js/jsx文件，env是根据环境自动变化的一个推荐包
              ]
            }
          }
        }
      ]
    }
  }

  3、在webpack5中配置babel-react的预设能力，让webpack支持打包jsx文件
  module.exports = {
    mode: 'production',
    module: {
      rules: [
        {
          // 用babel-loader来处理js/jsx文件，而非用webpack默认能力打包
          // 这样方便拓展更多能力
          test: /\.jsx?$/,
          exclude: /node_modules/, // 遇到node_modules文件不处理，因为这些文件都默认打包过
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                ['@babel/preset-env'], // 使用babel-loader预设能力处理js文件，env是根据环境自动变化的一个推荐包
                ['@babel/preset-react'], // 使用babel-react预设能力处理jsx文件
              ]
            }
          }
        }
      ]
    }
  }

  配置后，可以支持在入口文件中引入jsx，webpack在递归依赖分析的时候能够处理
  jsx文件如：
  const jsxDemo = () => {
    return (
      <div>jsxDemo</div>
    )
  }
  export { jsxDemo }

  4、通过引入插件来让webpack打包时，能使用Eslint来找到代码中的错误; 以及每次打包前清空output目录；
  const { CleanWebpackPlugin } = require('clean-webpack-plugin')
  const EslintPlugin = require('eslint-webpack-plugin')

  module.exports = {
    mode: 'production',
    plugins: [
      new CleanWebpackPlugin(), // emit阶段写文件前清空output目录，done阶段清空assets目录无用、过期的文件；
      new EslintPlugin( // 让webpack打包时，能使用Eslint去找到代码中的错误
        { extensions: ['.js', '.jsx'] }
      ), 
    ],
    module: {
      rules: [
        {
          // 用babel-loader来处理js/jsx文件，而非用webpack默认能力打包
          // 这样方便拓展更多能力
          test: /\.jsx?$/,
          exclude: /node_modules/, // 遇到node_modules文件不处理，因为这些文件都默认打包过
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                ['@babel/preset-env'], // 使用babel-loader预设能力处理js文件，env是根据环境自动变化的一个推荐包
                ['@babel/preset-react'], // 使用babel-react预设能力处理jsx文件
              ]
            }
          }
        }
      ]
    }
  }

  然后看eslint中的配置
  module.exports = {
    extends: ['react-app'], // 继承react官方配置规则
    rules: {
      'react/jsx-uses-react': [2], // 要在jsx文件里使用react，括号内：0 - 错误时不限制； 1 - 错误时警告；2 - 错误时报错；
      'react/react-in-jsx-scope': [2], // 要在jsx中import React from 'react'
    }
  }

  5、在webpack5中配置babel-typescript的预设能力，让webpack支持打包ts、tsx文件，并且配置Eslint代码检测；
  并配置alias，让js/ts代码中可以用@代替src；

  const EsLintPlugin = require('eslint-webpack-plugin');
  const { CleanWebpackPlugin } = require('clean-webpack-plugin');
  const path = require('path')

  module.exports = {
    mode: 'production',
    plugins: [
      new CleanWebpackPlugin(), // emit阶段写文件前清空output目录，done阶段清空assets目录无用、过期的文件；
      new EsLintPlugin( // 让webpack打包时，能使用Eslint去找到代码中的错误
        { extensions: ['.js', '.jsx', '.ts', '.tsx'] }
      ), 
    ],
    resolve: {
      alias: { // 若webpack发现@符号，则默认找到当前目录下src
        '@': path.resolve(__dirname, './src/')
      }
    },
    module: {
      rules: [
        {
          // 用babel-loader来处理js/jsx文件，而非用webpack默认能力打包
          // 这样方便拓展更多能力
          test: /\.[tj]sx?$/,
          exclude: /node_modules/, // 遇到node_modules文件不处理，因为这些文件都默认打包过
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                ['@babel/preset-env'], // 使用babel-loader预设能力处理js文件，env是根据环境自动变化的一个推荐包
                ['@babel/preset-react', { runtime: 'classic' }], // 使用babel的预设react配置，让webpack支持打包jsx
                ['@babel/preset-typescript'], // 使用babel-loader预设能力处理ts文件
              ]
            }
          }
        }
      ]
    }
  }
```