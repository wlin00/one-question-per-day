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

  // 注意， 这里开始判断是否是css -----------------------------------> runloaders
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
    在make阶段的buildModule事件里，webpack先通过执行runLoaders方法对非js文件进行处理，然后webpack借助第三方库Acorn处理js为AST（webpack的本职工作也是借助Acorn做JavaScriptParse），然后就可以对AST做traverse操作进行递归的依赖分析，一旦发现当前ast节点的type为‘ImportDeclaration’，就把这依赖的子文件添加到当前module的dependencies中并递归依赖分析；
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
      下面记录一些用到过的插件：
        插件1：imagemin-webpack-plugin 能力：每次文件打包写文件前，检测当前依赖是否是图片，若是的话对其优化压缩，返回新的优化图片。
        在 compiler.hooks.emit.tapAsync 的回调中执行插件能力（即监听emit阶段），遍历compilation.assets编译文件，若发现当前编译文件是图片则调用optimizeImage来优化图片。
        插件2: clean-webpack-plugin 能力：每次文件打包写文件前，清空output输出目录；打包结束后，在done的事件阶段删除所有没用到的文件如垃圾文件、临时文件。
        该插件会在compile.hooks.emit.tap 的回调中执行插件能力（即监听emit阶段），若本次编译没出错，则在写文件到硬盘之前，清空output输出目录的文件；
        在done阶段，compile.hooks.done.tap 的回调中，遍历assets文件目录，删除所有要用的文件之外的文件如临时文件、垃圾文件。
        插件3：ProvidePlugin 能力：帮用户全局引入某个依赖，让后续文件中不用再每次都引入。
        该插件会在compile.hooks.compilation.tap 的回调中执行插件能力（即监听compilation阶段），该插件在compilation阶段获取到nmf普通模块工厂，并继续监听其原型上的parse事件（监听代码转换为ast的parse阶段）；该插件会在代码借助Acorn库转化为ast的过程中执行能力 - 遍历的 definition 配置，然后为当前模块引入所有 expression 表达式里用过的但未引入的依赖；
        即一个全局注入引入依赖的能力，配置后就不用在每个文件import vue from 'vue'。
        插件4: MiniCssExtractPlugin，用于在生产环节单独提取css文件到独立的bundle。
        插件5: EslintPlugin，让webpack打包时，能对代码进行esLint的校验。
        插件6: HtmlWebpackPlugin，文件打包时，自动生成html页面。
        插件7: WorkboxWebpackPlugin，提供pwa能力：如果访问一个网站时，服务器挂掉时，本地可以利用缓存依然呈现出之前的页面。（PWA：借助 Service Worker、离线存储、后台同步等技术来提供离线处理能力，让我们的页面在访问一次以后，再次访问时能读取上一次的缓存）

  
```

六、写一个plugin的5个要点
```javascript
  1、向外导出一个JavaScript命名函数或一个类；
  2、在插件函数的原型 or 类的方法上定义一个apply，webpack默认作为该插件能力；
  3、监听一个webpack自身通过Tapable发布的事件（如compile.hooks.compilation.tap)；
  4、webpack内部实例的数据处理；
  5、功能完成后调用 webpack 提供的callback回调。
```

下面写一个插件，其功能是：生成一个叫做 filelist.md 的新文件；文件内容是记录本次构建的所有依赖文件的列表。
```typescript
  // FileListPlugin - 用于emit阶段输出文件时，额外输出一个记录了本次打包依赖文件的md
  class FileListPlugin {
    // 插件定义为一个类， 构造函数里可以获取配置项里的options
    constructor(options) {
    }
    // apply方法为webpack读取的入口方法
    apply(compiler) {
      // 监听webpack 输出文件的emit阶段 (这个阶段可以获取本次打包的编译compilation)
      compiler.hooks.emit.tapAsync('FileListPlugin', (compilation, callback) => {
        // create a header string for the generated file
        var fileList = 'In this build:\n\n'
        // loop through all compiled assets
        for (var filename in compilation.assets) {
          // compilation - 代表本次打包对应的‘编译’， key是模块对应名称
          // value 是一个存放其源代码的对象 { source: string, size: number }
          // 构造md 文件， 每隔一行记录一个本次打包的依赖文件
          fileList += '- ' + filename + '\n';
        }

        // Insert the list into the webpack build as a new file asset
        // 额外输出一个 filelist.md 文件
        compilation.assets['filelist.md'] = {
          source: function() { // 不用箭头函数， 为了让webpack的this读取到正确的函数作用域
            return fileList
          },
          size: function() {
            return fileList.length
          }
        }
        // plugin 最后执行回调通知weboack
        callback()
      })
    }
  }

  module.exports = FileListPlugin
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


  6、配置sass-loader -> css-loader -> style-loader, 来支持打包scss文件

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
        },
        { // 处理scss文件，loader处理顺序, sass-loader -> css-loader -> style-loader
          test: /\.s[ac]ss$/i, // 不区分大小写的匹配scss / sass
          use: ['sass-loader', 'css-loader', 'style-loader']
        }
      ]
    }
  }


  7、配置sass-loader, 自动为所有scss/sass文件引入某个文件，如scss-vars.scss 变量文件
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
        },
        { // 处理scss文件，loader处理顺序, sass-loader -> css-loader -> style-loader
          test: /\.s[ac]ss$/i, // 不区分大小写的匹配scss / sass
          use: ['style-loader', 'css-loader', {
            loader: 'sass-loader',
            options: {
              additionalData: `@import '~@/scss-vars.scss';`, // 全局注入scss-vars.scss
              sassOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          }]
        }
      ]
    }
  }


  8、配置css-loader, 支持js读取scss导出的变量
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
        },
        { // 处理scss文件，loader处理顺序, sass-loader -> css-loader -> style-loader
          test: /\.s[ac]ss$/i, // 不区分大小写的匹配scss / sass
          use: ['style-loader', {
            loader: 'css-loader', // 额外配置css-loader，将解析类型改为icss，从而支持js读取css导出变量
            options: {
              modules: {
                compileType: 'icss'
              }
            }
          }, {
            loader: 'sass-loader',
            options: {
              additionalData: `@import '~@/scss-vars.scss';`, // 全局注入scss-vars.scss
              sassOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          }]
        }
      ]
    }
  }

  scss文件中，导出一个对象便于js获取
  $color: red;
  :export {
    color: $color
  }


  9、配置 MinCssExtractPlugin 来在《生产环境》单独抽取css文件为独立的bundle；
  配置HtmlWebpackPlugin，来自动生成html页面；
  并且抽离公共函数useStyleLoader，作为统一处理css/scss/less文件的loader；
  代码：
  const EsLintPlugin = require('eslint-webpack-plugin');
  const { CleanWebpackPlugin } = require('clean-webpack-plugin');
  const MiniCssExtractPlugin = require('mini-css-extract-plugin')
  const HtmlWebpackPlugin = require('html-webpack-plugin')
  const path = require('path')
  const mode = 'production'

  // 简化代码 - 使用抽离函数统一处理css/scss/less
  const useStyleLoader = (...loader) => [
    // 参数1根据当前环境判断，若是生产环境，则单独提取css文件为额外bundle
    mode === 'production' ? MiniCssExtractPlugin.loader : 'style-loader',
    { //提取css文件为单独bundle，在emitAssets阶段多导出一个css
      loader: 'css-loader', // 额外配置css-loader，将解析类型改为icss，从而支持js读取css导出变量
      options: {
        modules: {
          compileType: 'icss'
        }
      }
    }, 
    ...loader // 根据入参loader处理
  ]

  module.exports = {
    mode,
    output: {
      filename: '[name].[contenthash].js', // 为输出的js添加hash，记得使用cleanWebpackPlugin 每次emit阶段清除dist
      path: path.resolve(__dirname, './dist')
    },
    plugins: [
      // emit阶段写文件前清空output目录，done阶段清空assets目录无用、过期的文件；
      new CleanWebpackPlugin(), 
      // 让webpack打包时，能使用Eslint去找到代码中的错误
      new EsLintPlugin( 
        { extensions: ['.js', '.jsx', '.ts', '.tsx'] }
      ), 
      // 自动生成html页面
      new HtmlWebpackPlugin(),
      // 单独提取css文件，需要在对应css-loader前将MiniCssExtractPlugin.loader替换style-loader
      mode === 'production' && new MiniCssExtractPlugin({ 
        filename: '[name].[contenthash].css' // 为css添加hash
      }), 
    ].filter(Boolean), // 过滤plugins中为false（即未开启）的选项，这样的写法给各个插件提供了开关的功能
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
        },
        { // 处理scss文件，loader处理顺序, sass-loader -> css-loader -> style-loader
          test: /\.s[ac]ss$/i, // 不区分大小写的匹配scss / sass
          use: useStyleLoader({
            loader: 'sass-loader',
            options: {
              additionalData: `@import '~@/scss-vars.scss';`, // 全局注入scss-vars.scss
              sassOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          })
        },
        { // 处理less文件，loader处理顺序, less-loader -> css-loader -> style-loader
          test: /\.less$/i, // 不区分大小写的匹配less
          use: useStyleLoader({
            loader: 'less-loader',
            options: {
              additionalData: `@import '~@/less-vars.less';`, // 全局注入less-vars.less
              lessOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          })
        }, 
        { // 处理css文件，loader处理顺序, css-loader -> style-loader
          test: /\.css$/i, // 不区分大小写的匹配scss / sass
          use: useStyleLoader()
        }
      ]
    }
  }

  10、webpack的一些优化配置：
    一、（runtimeChunk）单独打包运行时文件 runtime（即 webpack 为了让浏览器能执行我们打包后的代码，所需要的额外代码文件）
    作用：如果我们是单独打包 runtime， 在升级 webpack 版本等操作的时候（未动到打包文件源码），这样打包后的main.js不会改变，用户还是可以使用同一个main.js缓存，为用户节省带宽降低打开页面用时，提升体验。
    optimization: {
      runtimeChunk: 'single'
    }

    二、（usedExports）在开发环境，如果需要开启tree-shaking配置（生产环境默认开启）, 但webpack其实不会真正在开发环境去除不用的文件，而只是做标记
    optimization: {
      // tree-shaking：在es-module中，只去打包项目使用到的模块, 配置的话是开启usedExports: true;
      // 并在 package.json 的 sideEffects 中配置上不希望tree-shaking掉的依赖如："sideEffects": ['@babel/polly-fill'],这样避免一些不需要引入的包被shaking掉。
      // 若没有不想被shaking掉的包，则sideEffect设置为不去除css文件即可,如 "sideEffects": ["*.css"]
      usedExports: true
    }

    三、（splitChunks）单独打包 node_modules 中的依赖如react和vue：
      1、由于react、vue这些依赖不常升级改变；所以在编译的时候，为了能缓存之前的依赖，我们配置splitChunks来单独打包 node依赖
      2、为了用户缓存考虑，类似runtime单独打包，若只升级node依赖、而源文件代码未改变，用户还是可以使用同一个main.js缓存，节省带宽；
      optimization: {
        splitChunks: {
          cacheGroups: {
            vendor: {
              minSize: 0, // 不管这个node的包多小都单独打包
              test: /[\\/]node_modules[\\/]/, // 匹配/node_modules/ 和 \node_modules\
              name: 'vendor', // 单独打包输出到dist目录命名为 vendor.[hash]?.js
              chunks: 'all', // all 表示把来自node依赖的同步加载(initial)和异步加载(async)的都单独打包
            }
          }
        }
      }
    
    四、（moduleIds）固定模块id
      webpack打包时，会按一定顺序对模块编号；为保证用户能尽量缓存使用已经下载的没有变化的文件，则将没变化的模块id（文件名）固定；
      optimization: {
        moduleIds: 'deterministic',
      }
  
  11、多页面、多入口打包
    配置entry, 为各个入口文件配置前缀key
    entry: {
      index: './src/index.js',
      admin: './src/admin.js',
    }

    然后在HtmlWebpackPlugin中配置生成的页面所对应的入口文件
    plugins: [
      // 自动生成html页面，多页面配置时，按页面数量配置HtmlWebpackPlugin
      new HtmlWebpackPlugin({
        filename: 'index.html', // 生成的文件名
        chunks: ['index'], // 对应的入口文件
      }),
      new HtmlWebpackPlugin({
        filename: 'admin.html', // 生成的文件名
        chunks: ['admin'], // 对应的入口文件
      }),
    ].filter(Boolean)

    多页面配置后，记得在splitChunks中加入common chunks配置
    配置common属性，可以在多页面打包的情况下，将多个页面都用到的依赖单独打包，这样就不用每个页面都打包一次；
    splitChunks: {
      common: {
        priority: 5, // 区分打包优先级，如具体打到common chunks、vendor chunks（第三方包）哪个里面，
        minSize: 0, // 不管这个共同引入的包多小都单独打包,
        minChunks: 2, // 最少两个页面共同使用就独立打包，
        chunks: 'all', // all 表示把来自node依赖的同步加载(initial)和异步加载(async)的都单独打包
        name: 'common', // 单独打包输出到dist目录命名为 common.[hash]?.js
      }
    }

    12、优化任务：开启多进程打包（thread-loader）压缩js（terser-webpack-plugin）、压缩css（css-minimizer-webpack-plugin）;
      (1) 配置 thread-loader 来开启多进程加速打包：一般我们文件的js编译较慢，所以thread-loader 一般和 babel-loader 一起配置
      例如处理js的时候
        rules: [
          {
            // 用babel-loader来处理js/jsx文件，而非用webpack默认能力打包
            // 这样方便拓展更多能力
            test: /\.[tj]sx?$/,
            exclude: /node_modules/, // 遇到node_modules文件不处理，因为这些文件都默认打包过
            use: [
              'thread-loader', // 处理js/jsx/ts/tsx 时，开启多进程打包（这个 loader 位置之后的 loader 会放在一个单独的 worker 池， 每个worker具备600ms限制所以只在耗时loader开启）
              {
                loader: 'babel-loader',
                options: {
                  presets: [
                    ['@babel/preset-env'], // 使用babel-loader预设能力处理js文件，env是根据环境自动变化的一个推荐包
                    ['@babel/preset-react', { runtime: 'classic' }], // 使用babel的预设react配置，让webpack支持打包jsx
                    ['@babel/preset-typescript'], // 使用babel-loader预设能力处理ts文件
                  ]
                }
              }
            ]
          }]

      (2) 配置 terser 来压缩js体积，terser的一个新功能《编译期预计算》，如const day = 365 * 24 * 60， 而terser会帮我们进行《编译期预计算》，它会在编译的时候，会帮我们把一些可以计算的代码进行计算，让我们可以节省体积以及在运行的时候减少cpu使用。
      const TerserPlugin = require('terser-webpack-plugin') // 压缩js代码
      plugins: [
        // terser 压缩js/ts/jsx/tsx 代码
        new TerserPlugin({
          test: /\.[tj]sx?$/
        }),
      ]

      (3) 配置 css-minimizer-webpack-plugin 来压缩css体积, 本质上是代码的合并
      const CssMinimizerPlugin = require('css-minimizer-webpack-plugin') // 压缩css代码
      optimization: {
        minimizer: [
          new CssMinimizerPlugin()
        ]
      }
```

八、webpack 配置代码示例
```typescript
  const EsLintPlugin = require('eslint-webpack-plugin');
  const { CleanWebpackPlugin } = require('clean-webpack-plugin');
  const MiniCssExtractPlugin = require('mini-css-extract-plugin')
  const HtmlWebpackPlugin = require('html-webpack-plugin')
  const WorkboxWebpackPlugin = require('workbox-webpack-plugin') // 生产环境，开启pwa能力，对已打开的页面做离线缓存
  const TerserPlugin = require('terser-webpack-plugin') // 压缩js代码
  const CssMinimizerPlugin = require('css-minimizer-webpack-plugin') // 压缩css代码
  const path = require('path')
  const mode = 'production'
  // const mode = 'development'

  // 简化代码 - 使用抽离函数统一处理css/scss/less
  const useStyleLoader = (...loader) => [
    // 参数1根据当前环境判断，若是生产环境，则单独提取css文件为额外bundle
    mode === 'production' ? MiniCssExtractPlugin.loader : 'style-loader',
    { //提取css文件为单独bundle，在emitAssets阶段多导出一个css
      loader: 'css-loader', // 额外配置css-loader，将解析类型改为icss，从而支持js读取css导出变量
      options: {
        modules: {
          compileType: 'icss'
        }
      }
    }, 
    ...loader // 根据入参loader处理
  ]

  module.exports = {
    mode,
    entry: { // 多入口打包配置，index 和 admin 分别代表前端页面 和 后台页面的打包入口；（如果使用了多页面打包记得common chunks优化）
      index: './src/index.js',
      admin: './src/admin.js'
    }, 
    devServer: { 
      //webpack4 配置contentBase，将默认服务访问的目录改为dist打包目录
      // contentBase: path.join(__dirname, './dist')

      // webpack5.x 改为static配置来 - 默认服务器访问路径为dist目录
      static: './dist'
    },
    devtool: mode === 'development' ? 'eval-cheap-module-source-map' : 'cheap-module-source-map', // 不关心列信息 -> 获取经过loader处理后的es5代码 -> 每个模块用eval（）执行加速重构建 -> 报错信息精准；
    output: {
      filename: '[name].[contenthash].js', // 为输出的js添加hash，记得使用cleanWebpackPlugin 每次emit阶段清除dist
      path: path.resolve(__dirname, './dist')
    },
    // webpack优化配置，加快打包速度和体验等优化
    optimization: { // （tree-shaking；单独打包运行时文件、第三方依赖、公共模块；固定模块Id）
      // tree-shaking：在es-module中，只去打包项目使用的模块；
      // 如果需要在开发环境使用tree-shaking，（但webpack其实不会真正在开发环境去除不用的文件，而只是做标记），就配置usedExports: true
      // 并在 package.json 的 sideEffects 中配置上不希望tree-shaking掉的依赖,如 "sideEffects": ["@babel/polly-fill"] 这类没有向外部暴露变量而只是在window上添加对象的包；
      // 若没有不想被shaking掉的包，则sideEffect设置为不去除css文件即可,如 "sideEffects": ["*.css"]
      usedExports: true,

      // 单独打包运行时文件（即webpack让浏览器中能运行我们打包代码所需要的额外代码）
      // 好处是：如果我们是单独打包运行时文件，在升级webpack版本等操作时（未动到文件源码）这样main.js没有改变，可以让用户依然使用main.js的缓存，为用户节省带宽降低打开页面用时，提升体验。
      runtimeChunk: 'single', 

      // 单独打包 node_modules引入的依赖 如 import React from 'react';
      // 1、由于react、vue这些依赖不常升级改变；所以在编译的时候，为了能缓存之前的依赖，我们配置splitChunks来单独打包 node依赖
      // 2、为了用户缓存考虑，类似runtime单独打包，若只升级node依赖、而源文件代码未改变，用户还是可以使用同一个main.js缓存，节省带宽；
      // 3、配置common属性，可以在多页面打包的情况下，将多个页面都用到的依赖单独打包，这样就不用每个页面都打包一次；
      splitChunks: {
        cacheGroups: {
          vendor: {
            minSize: 0, // 不管这个node的包多小都单独打包
            test: /[\\/]node_modules[\\/]/, // 匹配/node_modules/ 和 \node_modules\
            name: 'vendors', // 单独打包输出到dist目录命名为 vendor.[hash]?.js
            chunks: 'all' // all 表示把来自node依赖的同步加载(initial)和异步加载(async)的都单独打包
          },
          common: {
            minSize: 0, // 不管这个共同引入的包多小都单独打包,
            minChunks: 2, // 最少两个页面共同使用就独立打包，
            chunks: 'all', // all 表示把来自node依赖的同步加载(initial)和异步加载(async)的都单独打包
            name: 'common', // 单独打包输出到dist目录命名为 common.[hash]?.js
          }
        }
      },

      // webpack打包时，会按一定顺序对模块编号；保证用户能尽量缓存使用已下载的没有变化的文件，则将没变化的模块id即文件名固定；
      moduleIds: 'deterministic', // 确定模块id

      // 压缩css 插件写在 optimization 的 minimizer 里
      minimizer: [
        new CssMinimizerPlugin()
      ]
    },
    plugins: [
      // emit阶段写文件前清空output目录，done阶段清空assets目录无用、过期的文件；
      new CleanWebpackPlugin(), 
      // 让webpack打包时，能使用Eslint去找到代码中的错误
      new EsLintPlugin( 
        { extensions: ['.js', '.jsx', '.ts', '.tsx'] }
      ), 
      // 单独提取css文件，需要在对应css-loader前将MiniCssExtractPlugin.loader替换style-loader
      mode === 'production' && new MiniCssExtractPlugin({ 
        filename: '[name].[contenthash].css' // 为css添加hash
      }),
      // pwa 利用service worker开启离线缓存能力
      mode === 'production' && new WorkboxWebpackPlugin.GenerateSW({
        clientsClaim: true,
        skipWaiting: true,
      }),
      // terser 压缩js/ts/jsx/tsx 代码
      new TerserPlugin({
        test: /\.[tj]sx?$/
      }),
      // 自动生成html页面, 多页面配置中，按页面数量配置HtmlWebpackPlugin
      new HtmlWebpackPlugin({
        filename: 'index.html',
        chunks: ['index'], // 索引到对应入口文件的前缀
      }),
      new HtmlWebpackPlugin({
        filename: 'admin.html',
        chunks: ['admin'], // 索引到对应入口文件的前缀
      }),
    ].filter(Boolean), // 过滤plugins中为false（即未开启）的选项，这样的写法给各个插件提供了开关的功能
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
          use: [
            'thread-loader', // 处理js/jsx/ts/tsx 时，开启多进程打包（这个 loader 位置之后的 loader 会放在一个单独的 worker 池， 每个worker具备600ms限制所以只在耗时loader开启）
            {
              loader: 'babel-loader',
              options: {
                presets: [
                  ['@babel/preset-env'], // 使用babel-loader预设能力处理js文件，env是根据环境自动变化的一个推荐包
                  ['@babel/preset-react', { runtime: 'classic' }], // 使用babel的预设react配置，让webpack支持打包jsx
                  ['@babel/preset-typescript'], // 使用babel-loader预设能力处理ts文件
                ]
              }
            }
          ]
        },
        { // 处理scss文件，loader处理顺序, sass-loader -> css-loader -> style-loader
          test: /\.s[ac]ss$/i, // 不区分大小写的匹配scss / sass
          use: useStyleLoader({
            loader: 'sass-loader',
            options: {
              additionalData: `@import '~@/scss-vars.scss';`, // 全局注入scss-vars.scss
              sassOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          })
        },
        { // 处理less文件，loader处理顺序, less-loader -> css-loader -> style-loader
          test: /\.less$/i, // 不区分大小写的匹配less
          use: useStyleLoader({
            loader: 'less-loader',
            options: {
              additionalData: `@import '~@/less-vars.less';`, // 全局注入less-vars.less
              lessOptions: { // 若不配置alias，引入文件基于当前目录__dirname
                includePaths: [__dirname]
              }
            }
          })
        }, 
        { // 处理css文件，loader处理顺序, css-loader -> style-loader
          test: /\.css$/i, // 不区分大小写的匹配scss / sass
          use: useStyleLoader()
        }
      ]
    }
  }
```