**题目1 webpack用来做什么的，原理是什么**：
```typescript
  webpack用于解决两个问题：
  1、支持浏览器运行包含import、export的代码（EsModule），可以把关键字转换成对应功能的函数；
  （注：现代浏览器可以通过<script type='module'>来支持import和export，而IE8～IE15等浏览器则不支持；并且这种方法会根据依赖加载所有文件，所以用webpack进行依赖分析、把关键字转换为普通代码，并打包为一个js）
  2、支持把多个文件打包成一个js
```

**题目2 loader**：
```typescript
  1、webpack自带的打包器只能处理js文件
  2、当我们想加载css/less/scss/ts/md等文件的时候，需要特定的loader来提供特殊的功能
  3、loader就是将不同类型的文件转换为可处理的js，再由第三方库Acorn进行JavascriptParser
```

**题目3 plugin的作用**：
```typescript
  plugin是webpack的拓展器，在webpack运行的生命周期中会有多个阶段的事件（基于Tapable作为事件中心的发布订阅架构），而插件就是可以对webpack的每一个阶段做监听，可以插入到每一个打包阶段，来实现各种功能从而丰富webpack的能力。
```


**题目4 webpack如何优化项目体积**：
```typescript
  1、插件配置：使用 terser-webpack-plugin 压缩JS代码
    const TerserPlugin = require('terser-webpack-plugin')
    // 配置webpack plugins
    module.exports = {
      plugins: [
        new TerserPlugin({
          test: /\.[tj]sx?$/ // 对.js、.ts、.jsx、.tsx文件做压缩
        })
      ]
    }
  2、优化配置
    css-minimizer-webpack-plugin 压缩css文件 
    & 运行时单独打包
    & Treeshaking 配置
    & splitChunks分包配置
    & 多页面打包，共同依赖只打一份
    const CssMinimizerPlugin = require('css-minimizer-webpack-plugin') 
    module.exports = {
      optimization: {
        // 压缩css插件
        minimizer: [
          new CssMinimizerPlugin()
        ],
        // 运行时单独打包（webpack让浏览器运行我们代码所需要的额外代码，单独打包）
        runtimeChunk: 'single'

        // tree-shaking：在es-module中，只去打包项目使用的模块
        // 若需要开发环境开启，则配置usedExports；可以在package.json 配置sideEffects，来配置不希望被 “shaking” 掉的依赖；如 "sideEffects": ["@babel/polly-fill"] 这类只是在window上添加对象而没有向外export变量的包
        // 一般，如果没有担心被shaking掉的包，就配置css不去除即可，如："sideEffects":["*.css"]
        usedExports: true,

        // webpack分包配置
        splitChunks: {
          // 1、由于react、vue这类依赖不常升级，所以在编译时，为了能缓存之前的依赖我们配置splitChunks分 包，来单独打包node依赖
          // 2、为了用户缓存考虑，类似runtime运行时单独打包，即如果这些单独打包的依赖有升级，但源文件代码没有变化，那么用户仍然可以使用同一个main.js的缓存来节省带宽；
          cacheGroups: {
            vendor: {
              minSize: 0, // 不管这个node_modules里的第三方包多大都单独打包
              test: /[\\/]node_modules[\\/]/, // 匹配 /node_modules/ 和 \node_modules\
              name: 'vendors', // 最后输出到dist的目录里，文件名为vendor.[hash]?.js
              chunks: 'all', // all 代表把来自node依赖的同步加载(initial) 和异步加载(async) 都单独打包， 默认值 'async'
            }
          }
          // 3、配置common属性，则可以支持在多页面打包的情况下，将多个页面能用到的依赖单独打包，这样就不用每个页面都打包一次
          common: {
            minSize: 0,
            minChunks: 2, // 最少两个页面共同使用就进行独立打包
            chunks: 'all',
            name: 'common'
          }
        }
      }
    }
```
