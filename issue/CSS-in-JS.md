## 1、模块化
  **模块化**：是指在解决一个复杂问题时，`自顶向下` 逐层把系统划分为若干子模块的过程，即把一个大的功能拆分成小功能。

  > **JS：** 

  ```
    而浏览器的Js之前没有模块的概念，全依靠不同的全局变量（命名空间）来隔离；直到后来 AMD、CMD、CommonJS、ESM 等规范出现，使得打包后的代码具备模块级的作用域 -> 组件开发。
  ```

  > **CSS的模块化主要有两类方案：**

  ```
    一、运行时通过命名规范隔离样式如：BEM，它是通过 .block__element--modifer 这种命名规范实现样式不冲突。
    二、编译时，自动转换CSS，为其添加上模块的唯一区分。
  ```

  > 编译时的方案常用的一般有两种，一是：scoped， 二是：css-modules；

  - scoped：是大家熟悉的 `Vue` 中的样式隔离方案，是由 `vue-loader` 所支持。例如在我们的 `webpack` 项目中，在编译时，`vue-loader` 会给元素添加 `data-v-xxx` 的属性，给css添加一个对应的属性选择器来实现，如：
    
  ```html
    <!-- 编译前 -->
    <style scoped>
      .red {
        color: red;
      }
    </style>
    <template>
      <div class="red">hi</div>
    </template>

    <!-- 编译后 -->
    <style>
      .red[data-v-a1234] {
        color: red;
      }
    </style>
    <template>
      <div class="red" data-v-a1234>hi</div>
    </template>
  ```
  优点是：用户无感知

  - css-modules：是 `css-loader` 中的样式隔离方案。它是通过编译的时候，修改选择器的名字为全局唯一的class，来实现css隔离。如：

  ```html
    <!-- 编译前 -->
    <style module>
      .red {
        color: red;
      }
    </style>
    <template>
      <div :class="$style.red">hi</div>
    </template>

    <!-- 编译后 -->
    <style>
      ._321HUIH321ddUHIsAd__1 {
        color: red;
      }
    </style>
    <template>
      <div class="_321HUIH321ddUHIsAd__1">hi</div>
    </template>
  ```

  优点是：无关框架，vue和react通用，用户需要用 `styles.xxx` 来引用编译后的名字。


## 2、CSS-in-JS
  CSS-in-JS：将应用的 CSS 样式写在JS文件里，从而使用一些JS特性如：模块声明、变量定义、函数调用、条件判断等特性。
