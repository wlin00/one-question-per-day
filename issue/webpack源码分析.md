### 前言
该文章，对webpack原理和源码进行分析

### 第一章 - AST、Babel和依赖

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