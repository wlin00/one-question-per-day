## 1、 关于VSCode插件：
由微软出的一款轻量级代码编辑器VSCode，是非常轻量级的IDE，它的许多强大功能都依靠插件来实现。由浏览器实现的VSCode，其插件也是基于前端的各种技术实现如HTML、JS。插件本身是一个vsix文件，用于拓展各种功能。

## 2、开发VSCode插件的第一步
我们可以从VSCode的[官方文档](https://code.visualstudio.com/docs/editor/userdefinedsnippets)发现，VSCode提供一个官方的脚手架，来让开发者能够进行插件的开发，我们只需要执行下面代码：
```NPM
npm install -g yo generator-code
```
即可全局安装脚手架，并且我们同时需要vsce来将我们的代码打包为vsix的文件。
```NPM
npm install -g vsce
```

## 3、利用脚手架搭建项目
利用yo创建项目
```NPM
yo code
```
执行后，选择插件类型为Code Snippets, 然后填写相应信息。
![image](https://user-images.githubusercontent.com/48883217/97102757-26d7db80-16e3-11eb-81eb-3f39c64da0d3.png)

## 4、开发Element-ui代码片段插件
Element-ui组件库，我们按照其组件的类型，可分为如下几类：
Basic基础、Form表单、Data数据、Notice通知、Navigation导航、Other其他类。
我们按这几个分类，创建对应的json文件。

![image](https://user-images.githubusercontent.com/48883217/97103285-77046d00-16e6-11eb-8708-849e3be42069.png)

> 创建好组件类型对应文件后，在package.json里通过contributes配置项来关联相应文件

![image](https://user-images.githubusercontent.com/48883217/97103357-d1053280-16e6-11eb-8ba8-4f5da04b6fa9.png)
通过vsce打包我们的代码,即可开发出扩展IDE功能的插件。

下面拿基础类型的组件el-button来做一个示范：
```JSON
{
  "el-button": {
    "prefix": "el-button",
    "body": [
       "<el-button type=\"${1|primary,success,warning,danger,info,text|}\" size=\"${2|small,large|}\">",
        "\t${TM_SELECTED_TEXT:${3:按钮文字}}",
        "</el-button>"
    ],
    "description": "https://element.eleme.cn/#/zh-CN/component/button"
  }
}
```
1：prefix表示这段代码片段的代号名
2：description里可以附属加上代码的描述信息，这里附带上了element的官网地址。
3：body配置里就是组件的具体代码，其中的${1|...}就表示了光标出现的顺序，如光标出现在type属性供用户选择，而用户选择后按下Tab切换后，即可切换到${2|...}所在的位置再次选择。
4：${TM_SELECTED_TEXT : ... } 里则可填写默认的文本字样。

上述JSON文件中，我们就完成了el-button的一个基础用法的代码片段，通过vsce打包，在VSCode插件选项上，选择安装本地插件，引入我们打包好的文件。

![image](https://user-images.githubusercontent.com/48883217/97103787-b7191f00-16e9-11eb-82ca-762bc5f64a67.png)

> 当我们在vue文件中敲el时，即可看见我们刚刚开发的代码片段插件

![image](https://user-images.githubusercontent.com/48883217/97103819-fa738d80-16e9-11eb-9e36-931b4827a980.png)
![image](https://user-images.githubusercontent.com/48883217/97103828-04958c00-16ea-11eb-8411-49f614819cc7.png)


[参考链接](https://juejin.im/post/6884964643400318983#heading-1)