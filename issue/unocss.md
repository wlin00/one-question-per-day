### 一、原子化css的演变历史
```ts
  1、首先出现的是原子化css方案是：TailwindCSS - 1.0，它带来了将css直接写在dom上的改革，但一开始也有不少弊端，最大的一点是它会在初始化时将所有可能用到的样式先提前写好，即没有做按需引入，所以一开始的css文件已经非常冗余；
  2、接着出现了WindiCSS，它则舍弃了一开始引入所有css的方式，而是对代码的html进行扫描，然后筛选出真正用到的css来做到按需引入；这种按需也可称为 JIT（just in time）；后续Tailwind.css的新版也借鉴了WindiCSS这种方式来完成了按需；
  3、隶属于WindiCSS团队的antfu，基于windi的思想开发了一套UnoCSS，它更加轻量并具备前者的优势特性；而下一代的WindiCSS也将把UnoCSS作为其引擎；
```

### 二、安装
```ts
  1、package.json 中安装unocss依赖如： "devDependencies": { "unocss": "0.45.30" }
  2、配置vite.config.ts，使用uno的vite插件：
    import { defineConfig } from 'vite'
    import react from '@vitejs/plugin-react'
    import Unocss from 'unocss/vite'
    export default defineConfig({
      base: '/',
      plugins: [
        Unocss(),
        react(),
      ]
    })
  3、添加uno.config.ts配置文件
    import {
      defineConfig, presetAttributify, presetIcons,
      presetTypography, presetUno, transformerAttributifyJsx
    } from 'unocss'
    export default defineConfig({
      theme: {
      },
      shortcuts: {
      },
      safelist: [],
      presets: [
        presetUno(),
        presetAttributify(),
        presetIcons({
          extraProperties: { 'display': 'inline-block', 'vertical-align': 'middle' },
        }),
        presetTypography(),
      ],
      transformers: [
        transformerAttributifyJsx()
      ]
    })
  4、配置shims.d.ts 为html声明更多属性，防止ts识别到html上不存在的属性报错
    import * as React from 'react'
    declare module 'react' {
      interface HTMLAttributes<T> extends AriaAttributes, DOMAttributes<T> {
        flex?: boolean
        relative?: boolean
        text?: string
        grid?: boolean
        before?: string
        after?: string
        shadow?: boolean
      }
    }
```

### 三、使用
```ts
  1、基础使用，给div添加1px的红色边框：
    import * as React from 'react'
    export const Test: React.FC = () => {
      return (
        <div b-1 b-red>test</div>
      )
    }
  2、声明flex布局三个div，并给他们赋予高度、flex-grow宽度、边框等
    import * as React from 'react'
    import { NavLink } from 'react-router-dom';

    export const Welcome3: React.FC = () => {
      return (
        <>
          <div flex> 
            <header className="w-20%" b-1 b-red h-100px>header</header>
            <main className="w-20%" flex-grow b-1 b-blue h-100px>main</main>
            <footer className="w-60%" flex-grow b-1 b-black h-100px>footer</footer>
          </div>
          <div b-1 b-red m-t-100px flex justify-center items-center>
            <NavLink to="/welcome/4">下一页</NavLink>
          </div>
        </>
      )
    }
```