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
