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

### 3、制作Vite Svg Sprites插件，来每次批量、一次性的加载svg矢量图
思路：
（1）基于 `svgstore` 和 `svgo`两个库，
  `svgstore`作为svg仓库
  `svgo`用来优化svg图片，如去除多余属性
（2）当插件的`load `方法被执行时，会先实例化svgstore创建一个store，然后读取配置项里的文件夹路径，并遍历这个路径，来拿到某个svg的`绝对路径` & `代码` & `文件名key`;
然后将调用`store.add`将svg添加至svg仓库。
（3）将仓库中的全量svg字符串取出来，并使用`svgo`进行矢量图优化，然后生成新的svg标签

```typescript
import path from 'path'
import fs from 'fs'
import store from 'svgstore' // 用于制作 SVG Sprites
import { optimize } from 'svgo' // 用于优化 SVG 文件

export const svgstore = (options = {}) => {
  const inputFolder = options.inputFolder || 'src/assets/icons'; // 插件默认处理的入口文件，存放svg图片的位置
  return {
    name: 'svgstore',
    resolveId(id) {
      if (id === '@svgstore') { // 绕过vscode校验，本意是在main.ts中impot 'svg_bundle.js'
        return 'svg_bundle.js'
      }
    },
    load(id) {
      if (id === 'svg_bundle.js') {
        const sprites = store(options);
        const iconsDir = path.resolve(inputFolder);
        for (const file of fs.readdirSync(iconsDir)) {
          const filepath = path.join(iconsDir, file); // 拿到某个svg的路径
          const svgid = path.parse(file).name
          let code = fs.readFileSync(filepath, { encoding: 'utf-8' });
          sprites.add(svgid, code) // 添加所有当前文件夹下的svg到svgstore
        }
        // 批处理svgstore，并优化svg，重新生成svg元素插入到body
        const { data: code } = optimize(sprites.toString({ inline: options.inline }), {
          plugins: [
            'cleanupAttrs', 'removeDoctype', 'removeComments', 'removeTitle', 'removeDesc', 
            'removeEmptyAttrs',
            { name: "removeAttrs", params: { attrs: "(data-name|data-xxx)" } }
          ]
        })
        return `const div = document.createElement('div')
          div.innerHTML = \`${code}\`
          const svg = div.getElementsByTagName('svg')[0]
          if (svg) {
            svg.style.position = 'absolute'
            svg.style.width = 0
            svg.style.height = 0
            svg.style.overflow = 'hidden'
            svg.setAttribute("aria-hidden", "true")
          }
          // listen dom ready event
          document.addEventListener('DOMContentLoaded', () => {
            if (document.body.firstChild) {
              document.body.insertBefore(div, document.body.firstChild)
            } else {
              document.body.appendChild(div)
            }
          })`
      }
    }
  }
}
```

### 4、useSwipe自定义hook
功能：根据用户传入的HTMLElement，监听该dom节点上的用户手指操作，动态计算当前滑动方向和偏移量
思路：
（1）：定义响应式变量 `start` 和 `end`
（2）：在组件挂载时，监听当前dom节点的touchstart、touchmove、touchend事件 - `滑动开始`和`滑动过程中`更新当前开始和结束的下标
（3）：组件卸载时，移除dom节点的事件监听
（4）：向外导出两个计算属性，用于动态计算当前滑动的位移和方向

代码：
```typescript
import { computed, onMounted, onUnmounted, ref, Ref } from "vue";

type Point = {
  x: number;
  y: number;
}

export const useSwipe = (element: Ref<HTMLElement | null>) => {
  // useSwipe 自定义hook，用于监听传入的dom节点的手指滑动事件；
  // 向外部返回一个计算属性的《方向》、《滑动结束标识符》、《滑动位移》等信息的对象
  const start = ref<Point>() // typeof [Point, undefined]
  const end = ref<Point>() // typeof [Point, undefined]
  const swiping = ref<boolean>(false)

  // 计算属性： 当前手指滑动发生时的x，y偏移（笛卡尔坐标）
  const distance = computed(() => {
    if (!start.value || !end.value) {
      return null
    }
    return {
      x: end.value.x - start.value.x,
      y: end.value.y - start.value.y,
    }
  })

  // 计算属性： 当前手指滑动发生时，根据结束下标 - 开始下标，来推导出滑动方向
  const direction = computed<string>(() => {
    if (!distance.value) {
      return ''
    }
    // 根据当前偏移的y和x的绝对值（即两个坐标轴上的投影），来判定当前方向是y轴还是x轴
    const { x, y } = distance.value
    if (Math.abs(x) > Math.abs(y)) {
      return x > 0 ? 'right' : 'left'
    } else {
      return y > 0 ? 'down' : 'up'
    }

  })

  // 手指滑动开始 - 记录开始坐标
  const handleTouchStart = (e: TouchEvent) => {
    start.value = {
      x: e.touches[0].clientX, // touches[0]获取到第一根手指的活动
      y: e.touches[0].clientY, // touches[0]获取到第一根手指的活动
    }
    swiping.value = true // 开始滑动标识符
  }

  // 手指滑动过程中
  const handleTouchMove = (e: TouchEvent) => {
    end.value = {
      x: e.touches[0].clientX,
      y: e.touches[0].clientY,
    }
  }

  // 手指滑动结束
  const handleTouchEnd = () => {
    swiping.value = false // 结束滑动标识符
  }
  
  onMounted(() => {
    if (!element.value) {
      return
    }
    element.value.addEventListener('touchstart', handleTouchStart)
    element.value.addEventListener('touchmove', handleTouchMove)
    element.value.addEventListener('touchend', handleTouchEnd)
  })

  onUnmounted(() => {
    if (!element.value) {
      return
    }
    element.value.removeEventListener('touchstart', handleTouchStart)
    element.value.removeEventListener('touchmove', handleTouchMove)
    element.value.removeEventListener('touchend', handleTouchEnd)
  })

  return {
    distance,
    direction,
    swiping,
  }
}
```