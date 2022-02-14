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
![Image text](https://raw.githubusercontent.com/wlin00/img/main/a1.png)

下面对转换后的代码进行理解
```
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