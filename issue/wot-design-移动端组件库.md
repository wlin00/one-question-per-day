# wot design 移动端组件库

**时间**

2018.4 - 2021.4



**情景**

京麦 app 客户端通过 H5 插件的方式做平台开放，由其他官方插件或第三方插件来补充商家端能力，为了保障统一的体验，设计部门除了一套设计规范，但市面上的组件的视觉和交互相差较大，各个插件的设计还原度较低，整体体验不统一。



**任务**

基于京麦 app 的设计规范，开发一个移动端 H5 组件库，一方面保障统一的用户体验，一方面提高开发效率。后面京麦 app 支持小程序开发，需要同时开发小程序版本的组件库。



**行动**

参考多家组件库的 api 设计（element-ui、vant），并根据自己部门 B 端的表单场景进行组件设计开发，并支持主题自定义、按需引用。小程序方面因为京东小程序的语法基本跟微信一样，考虑部门也有少部分微信小程序的需求，因此通过 gulp 构建工具来同时支持编译成京东小程序和微信小程序。



**结果**

20+个系统使用该组件库，整体的用户体验和开发效率有明显提升。



## 技术点

### 开发一个组件库，需要考虑什么问题

1. 组件按需加载
2. 自定义主题（主题变量，样式易覆盖）
3. 字体图标
4. 国际化
5. 单元测试
6. ESLint 和 commit lint
7. CI/CD （提 PR 的时候预览效果，推送 tag 的时候自动发布 release）
8. API 的易用性
9. 组件文档的完整性和易懂



### 组件按需加载功能

将每个组件的入口文件，打包为 commonjs 的方式，每个组件的样式文件也单独编译，再通过 babel 插件 [babel-plugin-import](https://www.npmjs.com/package/babel-plugin-import)，会自动将模块的引入编译为读取组件库包 lib 文件夹下对应的文件：

```javascript
import { Button } from 'wot-design';
```

编译为：

```javascript
const Button = requrie('wot-design/lib/button');
require('wot-design/lib/button/style');
```

**为什么打包为 commonjs 的方式？**

因为组件中有彼此互相引用的情况，如果打包为 umd，则会出现相同的代码打包多份的问题。



### 自定义主题

主要使用 scss 变量定义主题色、字号、尺寸等，样式单独为文件，不与组件代码放一块。

使用 BEM 规范定义类名，选择器层级较低，方便样式覆盖。



### 国际化

内置不同语言的文案，通过一个翻译函数，通过 `key` 去获取对应文案。

支持与 `vue-i18n` 结合，可自定义传入 `i18n` 翻译函数。



### 单元测试

使用 `Jest` 框架，再结合 `@vue/test-utils` 库（封装了一系列工具函数，如 mount 挂载组件、find 查找节点、trigger 触发事件），进行单元测试。



### CI/CD

使用 github 的 actions 功能：

1）提交 PR 时，进行 eslint 检测和构建检测，避免提交的代码运行有问题

2）在推送 tag 的时候，自动发布 npm，并发布 release 版本（使用社区的 actions 插件）

在提交 PR 的时候，再同时结合提供静态服务的第三方服务（Netlify），自动部署文档和 demo，方便进行 code review。

利用 github 的 web hooks，设置机器人，监听 Pull Request，自动进行标签打标（修改文件类型），以及 review 后自动合并代码。



### 部分组件的实现细节



#### 1. input-number 计数器的小数精度问题

见 [小数精度问题](../前端知识/Javascript/小数精度问题.md)



#### 2. 日历组件

**如何确定当前周数？**

每年的 1月1日，看其实在一周的哪个星期（getDay），在周日-周三，则表示是一年的第一周，否则表示为一年的最后一周。然后再根据当天与 1月1日的时间差计算周数差（一周7天），从而得到当天是第几周。



**如何展示每个月第一天在其应该在的位置？**

计算每个月第一天是哪一天，然后再与一周的起始日进行差值计算，算出中间偏差几个天数，然后对第一天设置 margin-left 偏移。



**滚动监听？**

小程序使用 `IntersectionObserver API`；H5 使用滚动事件监听，再结合 `getBoundingClientRect` 进行判断。

`IntersectionObserver` 提供了一种异步检测目标元素与祖先元素或 viewport 相交情况变化的方法。

可以用它来实现图片懒加载、无限滚动等。

具体见 [IntersectionObserver](../前端知识/Javascript/IntersectionObserver.md) 。



**日历相比其他移动端组件库的优势？**

因为京麦 app 是面向 B 端，有不少表单操作，其中就涉及到不少日期选择，而 pc 端本身用 element-ui 组件库，其日期组件功能很完整，有日范围、月范围选择（缺少周范围），支持日期时间范围。

因此移动端的需求也得给支持日范围、月范围、周范围、日期时间范围，与设计师同学参照了多款 app 的设计如携程、一嗨租车、神州租车、vant 组件库，最终设计一种比较好的交互形态出来（移动端屏幕小，因此要实现日期时间范围还是比较麻烦的）。



#### 3. picker 的滚筒效果？（其他同学开发）

设计师曾经看到 Nut UI 的日期选择器，觉得其滚筒效果体验比较好，因此我们一个同学对其进行研究，使用 rotate3d 进行设置，同时设置一个隐藏的平铺开的列表，用来进行滑动监听滚动位置，再将其反映到滚动列表，让其滚动起来。



#### 4. 图片裁剪组件？（其他同学开发）

使用一个方框圈定范围，通过手势，用 transform 对图片进行移动缩放，最后根据缩放大小、坐标位置进行换算，得出实际的裁剪位置，再使用 canvas 生成图片。



### 小程序组件库的开发问题



#### 为什么不使用 Taro 或者 uniapp 之类的框架

Taro 19年的时候出的，那会还有很多不确定性，且只是支持 React 写法，而我们团队是 Vue 技术栈，对其只是保持观望的状态。

而且那会 H5 组件库的组件也有将近 20 个左右了。

而京东小程序那会刚出来，Taro 尚不支持，就算支持了也不确定有什么坑在，我们团队算是第一个吃京东小程序的螃蟹，那会京东小程序框架也不成熟，遇到不少官方 bug，也是不断跟那边沟通，然后那边修复发版。

虽然那会京东小程序也有一些试点项目在使用，但从组件库的角度上，我们团队对其组件封装使用是有不少的推动的意义的。

现在京东小程序也完善了，Taro 基本也比较稳定了，也支持用 vue2 或者 vue3 来编写了，如果要支持 vue3，那么我们这次将会使用 Taro 来做 H5 和小程序的多端支持。



#### 小程序代码与 H5 代码的维护

小程序与 H5 的代码中，大约有 85% 可以复用，本身小程序框架就是类似于 vue 的写法，都用 template 来写 html，用声明式来写 JS 逻辑。借鉴于 Vant 这款优秀的组件库，它将小程序的 JS 框架包装了一层类 Vue 的声明方式：

1. 生命周期名称的转化：
   1. 小程序 `created` 转为 vue 的 `beforeCreate`
   2. 小程序 `attached` 转为 vue 的 `created`
   3. 小程序 `ready` 转为 vue 的 `mounted`
   4. 小程序 `detached` 转为 vue 的 `destroyed`
2. 小程序 `properties` 转为 vue 的 `props`

京东小程序一开始是基于微信的 1.9 版本的语法，不支持 behavior，因此实现了类似于 `mixin` 的操作，生命周期顺序触发（先顺序执行 mixin，再执行组件的生命周期），而像 data、properties 则进行对象合并。

因此，整体上，小程序的开发基本上也跟 vue 差不多。



#### clickoutside 问题处理

像 下拉菜单、popover 等组件，有弹出层，当点击弹出层以外的地方，应该将弹出层关闭，但小程序本身不允许操作 dom，无法像 H5 那样给 document 添加点击事件来进行判断，因此封装一个 clickoutside 的 mixin，将相关组件的实例保存到数组中，当点击某个弹出层时，将其他弹出层关闭，同时暴露统一关闭的方法，让开发者对页面最外层的 view 设置点击事件，以关闭所有弹出层。



#### 样式隔离问题

小程序组件与组件之间的样式是互相隔离的，因此像组件列表无法通过 nth-child 获取到某个子元素，需要通过 `relations` 将祖辈组件和儿孙组件进行关联，并计算儿孙组件的下标。



#### Toast 、MessageBox 类组件怎么挂载节点

因为小程序不让操作 dom，因此需要用户在页面中引入组件，然后在 JS 代码中通过暴露出的 Toast、MessageBox 方法，进行实例参数赋值调用。



#### 样式覆盖问题

因为小程序组件的样式具有隔离性，因此无法像 H5 那样直接通过选择器进行样式覆盖，小程序提供 externalClasses 允许



#### 动画组件 transition 的实现

模拟 vue 的 transition，设置 `enter`、`enter-to`、`leave`、`leave-to` 几个状态，每个状态分别设置不同的 class 类名。

`enter` 到 `enter-to` 状态，通过执行下一帧，进行切换（通过选择器的操作，模拟 `requestAnimationFrame` 实现）。

实现通过 `enter-class`、`enter-active-class`、`enter-to-class`、`leave-class`、`leave-active-class`、`leave-to-class` 设置自定义动画的类名。



### 不好与改进

1. **不好**： 目前维护两份代码，工作量较大，虽然大部分代码可以直接复用。

   **改进**： 可以考虑用 Taro 进行重构，并支持 vue3，但同时也得继续维护 vue2，毕竟还有项目在使用

2. **不好**： 目前团队主要为业务团队，而且业务激增，很少能有精力进行组件库的迭代

   **改进**： 如果重新回到一开始，会选择找其他前端团队共建的方式，毕竟也确实是在造轮子

