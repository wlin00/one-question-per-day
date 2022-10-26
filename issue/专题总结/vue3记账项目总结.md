### 1、ruby入门

**(1)构造函数**
```ruby
  class User
    def initialize(name) // 构造函数
      @name = name // @name 类似this，指向当前作用域
    end
    def hi(target)
      p "hi #{target}, I am #{@name}" // 类似模版字符串的使用，ruby中是用'#{}'
    end
  end

  user1 = User.new("Bob") // 创建一个User类的实例，并通过构造函数传参
  user1.hi("Alice")
```

**(2)数组操作，将一个数组的所有偶数项推入空数组**
```ruby
  // 普通写法、用select 或者 each 遍历数组
  resArr = []
  [1,2,3,4,5,6].select do |n|
    if n%2 == 0
      resArr << n
    end
  end
  p resArr  

  // 缩写：do something if something, [1,2,3,4,5,6] 可以简化为(1..6), (1..6)可以用 to_a 转为数组
  resArr = []
  [1,2,3,4,5,6].select do |n|
    resArr << n if n.even?
  end

  // 高效写法，不新建数组，而是原数组改动
  resArr = []
  (1..6).to_a.select! do |n|
    resArr << n if n % 2 == 0
  end  

  // 更简化写法，通过 &: 表示当前元素的方法
  p resArr  
  resArr = []
  [1,2,3,4,5,6].select(&:even?)
  p resArr  
```

### 2、vscode snippets
在vscode中通过 `command` + `shift` + `p` 唤起输入框，然后输入 `Snippets` ,进入对应代码json文件如 `typescriptreact.json` 来编辑代码片段
如下，制作一段 `vbt` 唤起的base代码用于vue3+tsx项目
```json
{
	// Place your snippets for typescriptreact here. Each snippet is defined under a snippet name and has a prefix, body and 
	// description. The prefix is what is used to trigger the snippet and the body will be expanded and inserted. Possible variables are:
	// $1, $2 for tab stops, $0 for the final cursor position, and ${1:label}, ${2:another} for placeholders. Placeholders with the 
	// same ids are connected.
	// Example:
	// "Print to console": {
	// 	"prefix": "log",
	// 	"body": [
	// 		"console.log('$1');",
	// 		"$2"
	// 	],
	// 	"description": "Log output to console"
  // }
  "Vue Component": {
    "prefix": "vbt", // vue3 + tsx base
    "body": [
      "import { defineComponent, ref } from 'vue';",
      "export const $1 = defineComponent({",
      "  setup: (props, context) => {",
      "    return () => (",
      "      <div>$2</div>",
      "    )",
      "  }",
      "})",
    ]
  }
}
```
