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
	"React.FC": {
		"prefix": "fc",
		"body": [
			"import * as React from 'react'",
			"",
			"export const $1: React.FC = () => {",
			"  return (",
			"    <div>$2</div>",
			"  )",
			"}"
		]
	},
  "Vue Component": {
    "prefix": "vbt", // vue3 + tsx base
    "body": [
      "import { defineComponent } from 'vue';",
      "export const $1 = defineComponent({",
      "  setup: (props, context) => {",
      "    return () => (",
      "      <div>$2</div>",
      "    )",
      "  }",
      "})",
    ]
  },
  "Import Module SCSS": {
    "prefix": "ims",
    "body": [
      "import s from './$1.module.scss';",
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

  interface Options { // 同步的钩子函数，在hook手指滑动开始前后插入事件
    beforeTouchStart?: (e: TouchEvent) => void,
    afterTouchStart?: (e: TouchEvent) => void,
    beforeTouchMove?: (e: TouchEvent) => void,
    afterTouchMove?: (e: TouchEvent) => void,
    beforeTouchEnd?: (e: TouchEvent) => void,
    afterTouchEnd?: (e: TouchEvent) => void,
  }

  export const useSwipe = (element: Ref<HTMLElement | undefined>, options: Options = { beforeTouchStart: (e: TouchEvent) => e.preventDefault() }) => { // 默认在touchstart前终止浏览器外框滑动事件
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
      options?.beforeTouchStart?.(e)
      start.value = {
        x: e.touches[0].clientX, // touches[0]获取到第一根手指的活动
        y: e.touches[0].clientY, // touches[0]获取到第一根手指的活动
      }
      swiping.value = true // 开始滑动标识符
      options?.afterTouchStart?.(e)
    }

    // 手指滑动过程中
    const handleTouchMove = (e: TouchEvent) => {
      options?.beforeTouchMove?.(e)
      if (!start.value) {
        return
      }
      end.value = {
        x: e.touches[0].clientX,
        y: e.touches[0].clientY,
      }
      options?.afterTouchMove?.(e)
    }

    // 手指滑动结束
    const handleTouchEnd = (e: TouchEvent) => {
      options?.beforeTouchEnd?.(e)
      swiping.value = false // 结束滑动标识符
      options?.afterTouchEnd?.(e)
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

**使用**
实现效果：`在手指滑动屏幕某个div时，进行路由的动画切换`
```tsx
  import { useSwipe } from '@/utils'
  import { useRoute, useRouter } from 'vue-router'
  // setup
  export const Wrapper = defineComponent({
    setup: (props, context) => {
      const main = ref<HTMLElement>()
      const { swiping, direction } = useSwipe(main, {
        beforeTouchStart: (e: TouchEvent) => { // 阻止浏览器默认滑动事件
          e.preventDefault()
        }
      })
      const push = throttle(() => {
        if (route.name === 'page1') {
          router.push('/page2')
        }
      }, 500)
      watchEffect(() => {
        if (swiping.value && direction.value === 'left') {
          push() // 判断当前手指左滑，屏幕切换到下一屏
        }
      })
      return () => <div ref={main}><RouterView/></div>
    }
  })
```

### 5、用到的http状态码
```ts
200 请求成功
422 请求body参数有误
429 请求过于频繁，可用于60秒发送一次的验证码场景
```


### 6、 笔记：我的Rails + Vue3/React + postgresql 的全栈应用

# 一、环境配置
  1、初始化数据库,在mac环境，执行下面代码初始化创建数据库，以`db-for-mangosteen`为主键key，并且关联到网络`network1`
  ```rb
    docker run -d      --name db-for-mangosteen      -e POSTGRES_USER=mangosteen      -e POSTGRES_PASSWORD=123456      -e POSTGRES_DB=mangosteen_dev      -e PGDATA=/var/lib/postgresql/data/pgdata      -v mangosteen-data:/var/lib/postgresql/data      --network=network1      postgres:14
  ```

# 二、部署配置
  1、持久化云服务器远程连接的`超时时长`：
    命令行输入 `vi /etc/ssh/sshd_config`，然后找到 `ClientAliveInterval` 修改为300，表示最大超时时长为5分钟, 找到 `ClientAliveCountMax` 修改为5，表示最大连接数为5

  2、ubuntu安装`docker`， `root` 用户权限下
  ```ts
    (1) 进入网站：https://docs.docker.com/engine/install/ubuntu/
    (2) 云服务器更新apt-get: apt-get update
    (3) 安装其他依赖：
      apt-get install \
      ca-certificates \
      curl \
      gnupg \
      lsb-release
    (4) Add Docker’s `official GPG key`:
      mkdir -p /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    (5) 开启 `docker` 仓库 :
      echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    (6) 安装docker:
      apt-get update  
      apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    (7) 验证docker安装成功
      docker --version
      docker run hello-world
  ```

  3、为当前项目和docker添加一个专属用户，而非一直使用`root`，方便后续开发：
  ```ts
    (1) adduser mangosteen, 创建一个专属用户 mangosteen 及其分组，
    (2) 将mongosteen用户添加到docker组： usermod -a -G docker mangosteen
    (3) 然后将root的 ~/.ssh/authroized_keys, 复制到mangosteen用户的home/.ssh目录, 并且将复制文件的权限交接给mangosteen用户：
      - 先进入 /home/mangosteen 创建.ssh文件夹，cd /home/mangosteen | mkdir .ssh
      - 然后进行文件拷贝：cp ~/.ssh/authroized_keys /home/mangosteen/.ssh
      - 文件交接，在/home/mangosteen目录下，交接.ssh权限给mangosteen：chown -R mangosteen:mangosteen .ssh
  ```

  4、使用 `docker` 部署思路 
  `后端部署`步骤
  ```ts
    (1) 云服务器准备一个新用户
    (2) 云服务器上安装docker
    (3) 上传本地的Dockerfile 和 源代码
    (4) 用Dockerfile构建运行环境
    (5) 在运行环境里运行源代码
    (6) 用 Nginx 做转发
  ```
  `后端版本更新`步骤
  ```ts
    (1) 上传新Dockerfile
    (2) 上传新的源代码
    (3) 用Dockerfile构建运行环境
    (4) 在新的运行环境里运行新的源代码
    (5) 用 Nginx 做转发
  ```
  `前端部署`步骤
  ```ts
    (1) 将静态资源上传CDN，代码中引用路径也改为cdn地址
    (2) 将项目打包上传到云服务器的nginx/html目录，并进行反向代理和gzip压缩等，需要分流考虑负载均衡
  ```

  5、部署当前应用到 `宿主机（mac）`步骤
  ```ts
    (1) 增加 pack_for_host.sh、setup_host.sh、host.Dockerfile等文件
    (2) 更改 pack_for_host.sh 执行权限： chmod +x bin/pack_for_host.sh
    (3) 执行 pack_for_host.sh 来将应用源代码打包至宿主环境
  ```

  6、执行代码 - 宿主机部署
  ```ts
    // （1）linux里打包源代码到宿主环境
    sh bin/pack_for_host.sh
    // （2）宿主机VsCode运行setup_host.sh 并传递DB_HOST、DB_PASSWORD、RAILS_MASTER_KEY 来启动外部容器并构建dockerFile的《生产环境》
    DB_HOST=db-for-mangosteen DB_PASSWORD=123456 RAILS_MASTER_KEY=2617e5cf2af9fc0108db38b42b470b80 mangosteen_deploy/setup_host.sh
    // （3）创建生产环境数据库
    docker exec -it mangosteen-prod-1 bin/rails db:create db:migrate
    // （4）宿主机部署的 host.DockerFile，用于构建docker镜像
    FROM ruby:3.0.0
    ENV RAILS_ENV production
    RUN mkdir /mangosteen
    RUN bundle config mirror.https://rubygems.org https://gems.ruby-china.com
    WORKDIR /mangosteen
    ADD mangosteen-*.tar.gz ./
    RUN bundle config set --local without 'development test'
    RUN bundle install
    ENTRYPOINT bundle exec puma
  ```

  7、执行代码 - 自动化云服务器部署， 先尝试docker中链接云服务器：ssh mangosteen@47.94.212.148
  ```sh
    # （1）linux里打包源代码到云服务器（scp -> ssh cp），然后在云服务器构建 Dockerfile出生产环境，并写入生产环境密钥
    sh bin/pack_for_remote.sh

    # （2）pack_for_remote.sh, 解析
    # 注意修改 user 和 ip
    user=mangosteen # 写好环境变量：ssh用户名、ip地址
    ip=47.94.212.148
    time=$(date +'%Y%m%d-%H%M%S')
    dist=tmp/mangosteen-$time.tar.gz
    current_dir=$(dirname $0)
    deploy_dir=/home/$user/deploys/$time
    gemfile=$current_dir/../Gemfile
    gemfile_lock=$current_dir/../Gemfile.lock
    vendor_cache_dir=$current_dir/../vendor/cache
    function title { # log 函数
      echo 
      echo "###############################################################################"
      echo "## $1"
      echo "###############################################################################" 
      echo 
    }
    yes | rm tmp/mangosteen-*.tar.gz;  # 删除之前的源代码压缩包
    title '打包源代码为压缩文件'
    bundle cache
    tar --exclude="tmp/cache/*" -czv -f $dist * # 压缩源代码为tar文件，可被docker构建时的ADD命令解压缩
    title '创建远程目录'
    ssh $user@$ip "mkdir -p $deploy_dir/vendor"
    title '上传压缩文件'
    scp $dist $user@$ip:$deploy_dir/ # ssh cp
    scp $gemfile $user@$ip:$deploy_dir/
    scp $gemfile_lock $user@$ip:$deploy_dir/
    scp -r $vendor_cache_dir $user@$ip:$deploy_dir/vendor/
    title '上传 Dockerfile'
    scp $current_dir/../config/host.Dockerfile $user@$ip:$deploy_dir/Dockerfile # 核心文件上传
    title '上传 setup 脚本'
    scp $current_dir/setup_remote.sh $user@$ip:$deploy_dir/
    title '上传版本号'
    ssh $user@$ip "echo $time > $deploy_dir/version"
    title '执行远程脚本'
    ssh $user@$ip "export version=$time; /bin/bash $deploy_dir/setup_remote.sh" # 跑完pack_for_remote.sh 继续跑setup_remote.sh 来构建docker，然后在开发环境执行源代码 以及初始化/更新数据库

    # （3）setup_remote.sh, 解析
    # 环境变量初始化
    user=mangosteen
    root=/home/$user/deploys/$version
    container_name=mangosteen-prod-1
    db_container_name=db-for-mangosteen-production
    DB_HOST=db-for-mangosteen-production
    RAILS_MASTER_KEY=2617e5cf2af9fc0108db38b42b470b80
    DB_PASSWORD=123456
    function set_env { # 可供输入参数
      name=$1
      while [ -z "${!name}" ]; do
        echo "> 请输入 $name:"
        read $name
        sed -i "1s/^/export $name=${!name}\n/" ~/.bashrc
        echo "${name} 已保存至 ~/.bashrc"
      done
    }
    function title { # log function
      echo 
      echo "###############################################################################"
      echo "## $1"
      echo "###############################################################################" 
      echo 
    }
    title '创建数据库'
    if [ "$(docker ps -aq -f name=^${DB_HOST}$)" ]; then
      echo '已存在数据库'
    else # 创建数据库
      docker run -d --name $DB_HOST \
                --network=network2 \
                -e POSTGRES_USER=mangosteen \
                -e POSTGRES_DB=mangosteen_production \
                -e POSTGRES_PASSWORD=$DB_PASSWORD \
                -e PGDATA=/var/lib/postgresql/data/pgdata \
                -v mangosteen-data:/var/lib/postgresql/data \
                postgres:14
      echo '创建成功'
    fi
    title 'docker build'
    docker build $root -t mangosteen:$version  # 构建docker镜像
    if [ "$(docker ps -aq -f name=^mangosteen-prod-1$)" ]; then
      title 'docker rm'
      docker rm -f $container_name
    fi
    title 'docker run'
    docker run -d -p 3000:3000 \ # docker运行
              --network=network2 \ 
              --name=$container_name \
              -e DB_HOST=$DB_HOST \
              -e DB_PASSWORD=$DB_PASSWORD \
              -e RAILS_MASTER_KEY=$RAILS_MASTER_KEY \
              mangosteen:$version

    echo
    echo "是否要更新数据库？[y/N]"
    read ans
    case $ans in # 生产环境数据库同步
        y|Y|1  )  echo "yes"; title '更新数据库'; docker exec $container_name bin/rails db:create db:migrate ;;
        n|N|2  )  echo "no" ;;
        ""     )  echo "no" ;;
    esac

    title '全部执行完毕'

    # （4）云服务器部署的 host.DockerFile，用于docker build
    FROM ruby:3.0.0
    ENV RAILS_ENV production
    RUN mkdir /mangosteen
    RUN bundle config mirror.https://rubygems.org https://gems.ruby-china.com
    WORKDIR /mangosteen
    ADD Gemfile /mangosteen
    ADD Gemfile.lock /mangosteen
    ADD vendor/cache /mangosteen/vendor/cache
    RUN bundle config set --local without 'development test'
    RUN bundle install --local
    ADD mangosteen-*.tar.gz ./
    ENTRYPOINT bundle exec puma

    # (5) 若更新数据库，遇到《Error response from daemon》则表示需要重启docker
    docker restart db-for-mangosteen-production # db-for-mangosteen-production 为当前环境的db_container_name
  ```


# 三、rails 密钥管理 - 开发环境和生产环节各有一个128位的密钥用于对称加密，来让应用具备安全性
  1、创建开发环境的master.key密钥，rails会对应创建加密好后的密文.enc，并复制放在临时文件中的密钥 `secret_key_base`
  ```ts
    // 创建开发环境密钥 & 密文，来获取开发环境权限
    rm config/credentials.yml.enc
    EDITOR="code --wait" bin/rails credentials:edit 
  ```
  2、创建生产环境密钥key，然后将复制的开发环境密钥替换给当前密钥
  ```ts
    EDITOR="code --wait" bin/rails credentials:edit --environment production
  ```
  3、重新生成一次开发环境密钥
  ```ts
    EDITOR="code --wait" bin/rails credentials:edit 
  ```
  4、如何查看开发/生产环境的密钥：
  ```rb
    # （1）打开开发环境的控制台
    bin/rails console
    # 使用开发环境的密钥，查看开发环境密文（获取所有加密信息的明文）
    Rails.application.credentials.config

    # （2）打开生产环境的控制台
    RAILS_ENV=production bin/rails console
    # 使用生产环境的密钥，查看生产环境密文（获取所有加密信息的明文）
    Rails.application.credentials.config
  ```

# 四、常用rails操作
  1、建表 - 建立`postgresql`中的一个`model模型`如User，执行后会生成两个文件：
    一是：class构造函数`user.rb` ，可用于定义一些字段约束，如`email字段必填校验`等；
    二是：用于直接修改数据库的`migrate`文件 `create_users.rb`，可以在此定义主键，或增加`长度limit`等校验
    ```rb
      bin/rails g model user email:string name:string
    ```

  2、同步模型的改动到数据库
  ```rb
    bin/rails db:migrate
  ```
  
  3、撤销上次更新操作 & 给数据库添加字段
  ```rb
    # 撤销操作
    bin/rails db:rollback step=1
    # 给某个表添加字段
    bin/rails g migration AddKindToItem
    # 若添加一个枚举字段，需要在对应字段的model文件添加：
    enum kind: { expenses: 1, income: 2 }
    validates :kind, presence: true
    # 并且编辑镜像文件《AddKindToItem》：
    def change
      add_column :items, :kind, :integer, default: 1, null: false
    end
  ```

  4、创建某个表如User的`controller`, 可以初始化方法如show、create， 具体的`api逻辑`在此实现
  ```rb
    bin/rails g controller users show create
  ```
  若要创建 `嵌套路由` 中的controller，则是执行：
  ```rb
    bin/rails g controller items Api::V1::Items # 注意路径 + 控制器名都要首字母大写
  ```

  5、使用`curl`模拟请求，调试接口
  ```rb
    # post 请求 - 保存用户接口
    curl -X POST http://127.0.0.1:3000/users # 可选择添加 -v 来查看http 状态码
    # get 请求，按ID查询接口
    curl http://127.0.0.1:3000/users/1
    # post 路由中调用
    curl -X POST http://127.0.0.1:3000/api/v1/validation_codes -H "Content-Type: application/json" -d '{"email":"wlin0z@163.com"}'
    # 部署云服务器后， 用curl模拟验证码请求（因这个请求不需要jwt鉴权）
    curl -X POST  http://47.94.212.148:3000/api/v1/validation_codes -H "Content-Type: application/json" -d '{"email":"wlin0z@163.com"}'
    # 手动在bin/rails console中创建记录，如创建一条验证码, 并测试发送邮件功能
    validation_code = ValidationCode.new email: 'wlin0z@163.com', kind: 'sign_in'
    validation_code.save
    UserMailer.welcome_email('wlin0z@163.com').deliver!
  ```
  
  6、mac环境重启 `db`
  ```rb
    docker start db-for-mangosteen
  ```

  7、加速bundle instal, 切换到ruby-china镜像
  ```rb
   bundle config mirror.https://rubygems.org https://gems.ruby-china.com
  ```

  8、全局配置kaminari
  ```rb
    bin/rails g kaminari:config
  ```

  9、使用 `kaminari` 分页查询demo
  ```rb
    # 查询账单记录
    items = Item.page(params[:page]).per(10) # 分页大小为10
    render json: {
      resource: items,
      pageInfo: {
        pageNo: params[:page],
        pageSize: 10,
        totalCount: Item.count
      }
    }
  ```

  10、安装单元测试框架 `rspec-rails` & 创建`测试环境数据库` 和 `建表`
  ```rb
    # 在Gemfile中
    group :development, :test do
      gem 'rspec-rails', '~> 5.0.0'
    end
    # 然后安装依赖
    bundle install --verbose
    # 然后生成rspec.rb
    bin/rails generate rspec:install
    # 创建测试环境数据库 （先在database.yml) 中完善测试数据库的用户名、pwd、host等信息
    RAILS_ENV=test bin/rails db:create
    # 创建测试环境表，只需要将现有的db/migrate同步测试数据库
    RAILS_ENV=test bin/rails db:migrate
  ```

  11、测试model
  ```rb
    bin/rails generate rspec:model items
  ```

  12、测试controller - 请求测试
  ```rb
    # 测试 账单记录controller
    bin/rails generate rspec:request items
    # 测试 验证码controller
    bin/rails generate rspec:request validation_codes
  ```

  13、执行单测
  ```rb
    bundle exec rspec
  ```

  14、rails借助 `SecureRandom` 生成一个`真随机`的`六位`数字验证码
  ```rb
  code = SecureRandom.random_number.to_s[2..7] # 生成一个安全真随机数，本质上是一个小数(0.3213213213...)，转化为字符串，并截取小数点后的1-6位，作为当前随机6位验证码
  ```

  15、使用Rails的 `Action Mailer` 模块 结合qq邮箱的 `第三方授权secret` 发邮件
  ```rb
    # (1) 创建rails的 mailer 模块
    bin/rails generate mailer User
    # (2) 进入/app/mailers/application_mailer.rb 在 ApplicationMailer 的类中，写好默认的邮递员配置
    class ApplicationMailer < ActionMailer::Base
      default from "616294069@qq.com"
      layout "mailer"
    end
    # (3) 进入/app/mailers/user_mailer.rb, 配置发送邮件的参数信息
    class UserMailer < ApplicationMailer
      def welcome_email(code)
        @code = code # 将入参的验证码传给@code，即可展示在html中
        mail(to: "wlin0z@163.com", subject: '请查收您的验证码')
      end
    end
    # (4) 进入/app/views/user_mailer 目录，新建邮件视图 welcome_email.html.erb
    <!DOCTYPE html>
    <html>
      <head>
        <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
      </head>
      <body>
      您的验证码为 <%= @code %>
      </body>
    </html> 
    # (5) 去qq邮箱，拿到个人账号的第三方授权密钥，然后将这个密钥添加到 rails的密钥管理的 《email_password》字段； 
    EDITOR="code --wait" bin/rails credentials:edit 
    # 然后修改 /config/environments/development.rb 添加对应密钥信息
    config.action_mailer.raise_delivery_errors = true
    config.action_mailer.delivery_method = :smtp
    config.action_mailer.smtp_settings = {
      address:              'smtp.qq.com',
      port:                 587,
      domain:               'smtp.qq.com',
      user_name:            '616294069@qq.com',
      password:             Rails.application.credentials.email_password, # 拿到当前邮递员的qq邮箱第三方授权密钥
      authentication:       'plain',
      enable_starttls_auto: true,
      open_timeout:         10,
      read_timeout:         10
    }
    # (6) 可以在rails console中测试发邮件功能
    bin/rails console
    UserMailer.welcome_email('123456').deliver # 也可发送六位真随机数 SecureRandom.random_number.to_s[2..7] 
  ```

  16、TDD 测试驱动开发, 开发完api ， 借助 `rspec_api_documentation` 生成api文档
  ```rb
    # (1) 添加《rspec_api_documentation》依赖, 并安装
    bundle install
    # (2) 根据官方文档创建文件目录, 写入测试示例
    mkdir spec/acceptance
    code spec/acceptance/order_spec.rb
    # (3) 生成文档
    bin/rake docs:generate 
    # (4) http-server 查看文档
    npx http-server doc/api/.
  ```

  17、`Jwt`相关笔记
  (1) jwt 定义 - json web token，可理解为把uid加密后的一个字符串
  即 `base64（header) + '.' + base64（payload) + '.' + base64（signature) + '.'` 的加密字符串的拼接
  ```ts
    jwt 由三个部分组成
     - Header：表示当前加密算法如HS256、和type类型如JWT
     - payload：表示当前的json，可包含用户信息如loggedInas：'admin', 可包含用户uid
     - signature：通过加密算法，来处理（私钥，base64(header), base64(payload))的集合，服务器可以对签名进行解密然后验证签名内的uid和签名外的是否一致，来确保jwt没有被篡改
  ```

  (2) 前后端使用 `jwt` 进行 `登录鉴权` 
  ```ts
    - 前端登录后，后端返回jwt，前端将jwt存储在 localstorage 里并在axios请求头进行拦截，在每一个请求头里添加jwt（一般是Authorization的字段）
    - 后端如果发现请求头中有authorization字段，就解密jwt，验证方式：服务端在请求头获取到jwt，拿到base64(header)+base64(paload)+base64(signature)，signature部分本质上是对前两部分的一个hs256的签名签名算法，而验证的方式是：再次对header和paload进行签名，比对前后的签名是否一致
  ```  
   
  (3) jwt 实例的`结构解析`
  下面是一个 jwt示例
  ```ts
    const jwt = 'eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0fQ.e8H0uo4CFXbcva_sJlv-dmo6jmiMzUvR35pyWZo7gG0'
    // 分别使用 反base64（window.atob)方法 对前两部分header、payload进行转义
    window.atob('eyJhbGciOiJIUzI1NiJ9') // '{"alg":"HS256"}'
    window.atob('eyJ1c2VyX2lkIjo0fQ') // '{"user_id":4}'
    // 而第三部分需要用当前HS256 对称加密算法（全称是：是HMAC+SHA256）结合密钥进行解密，解密结果是前两部分的一个整合；可以对比内外的user_id来判断jwt是否经过篡改
  ```

  (4) `jwt`的加密encode和解密decode用法
  ```rb
    #encode - 生成token
    payload = { user_id: user.id }
    # 传入载荷、密钥、加密算法：对称加密Hmac256 来生成jwt加密字符串
    token = JWT.encode payload, Rails.application.credentials.hmac_secret, 'HS256' 
    render status: :ok, json: { jwt: token }

    #decode - 获取请求头中的jwt，解密出jwt前两部分header和payload组成的对象数组([payload, header])
    header = request.headers['Authorization']
    jwt = header.split(' ')[1] rescue '' # rescue等同于try-catch
    payload = JWT.decode jwt, Rails.application.credentials.hmac_secret, true, { algorithm: 'HS256' } rescue nil
    return head 400 if payload.nil? # 如果当前解密发现jwt错误，返回400
    user_id = payload[0]['user_id'] rescue nil
    user = User.find user_id
    return head 404 if user.nil?
    render json: { resource: user }     
  ```

  (5) 将jwt的decode抽离成rails的 `中间件` 
  ```rb
    - 修改application.rb
      # 引入自定义的jwt中间件，用于route -> controller之间的jwt处理，会提取当前登录用户的user_id
      require_relative "../lib/auto_jwt" 
      # class Application 中，使用中间件
      config.middleware.use AutoJwt
    
    - 实现auto_jwt.rb
      class AutoJwt
        def initialize(app)
        end
        def call(env)
        end
      end
  ```  


  18、`登录接口`相关笔记
  (1) rails创建 `session_controller`：
  ```rb
    bin/rails g controller api/v1/sessions_controller # controller前缀为复数
  ```
  (2) 编写登录接口前, 先编写登录接口的测试用例，测试驱动开发
  ```rb
    # spec/requests/api/v1/sessions_spec.rb
    require 'rails_helper'
    RSpec.describe "Api::V1::Sessions", type: :request do
      describe "POST /api/v1/session" do
        it "can create a session" do # 期望有User账号后，能进行会话登录，登录后状态码200 & 响应体中有key为jwt & value为string的字段
          User.create email: 'wlin0z@163.com'
          post '/api/v1/session', params: { email: 'wlin0z@163.com', code: '123456' } # 模拟发送登录（创建会话）请求
          expect(response).to have_http_status(200)
          json = JSON.parse response.body
          expect(json['jwt']).to be_a(String) # 期望响应体的jwt字段是个string，若期望jwt为null可以写成 .to be_nil
        end
      end
    end
  ```
  (3) 编写登录接口，即创建会话接口 `create` 
  ```rb
    require 'jwt'
    class Api::V1::SessionsController < ApplicationController
    def create
      # 若当前是测试环境，验证码code固定为 '123456'
      if Rails.env.test?
        return status: :unauthorized if params[:code] != '123456'
      end
      # 若前当非测试环境，需先进行《当前会话前是否发送验证码》校验
      if !Rails.env.test?
        canSignInFlag = ValidationCodes.exists? email: params[:email], code: params[:code], used_at: nil
        return render status: :unauthorized unless canSignInFlag # 若不能登录则return
      end
      # 创建会话，校验当前user是否存在于User表中（防止错误删除了User表)
      user = User.find_by_email params[:email]
      if user.nil?
        return render status: :not_found, json: {errors: '用户不存在'}
      end
      # 登录校验成功，创建响应数据；载荷里放入uid
      # 在rails密钥管理中写入 hmac的密钥 -> hmac_secret: 'wlin$ecretK3y5050'
      payload = { user_id: user.id }
      # 创建JWT，定义header、payload、signature，传入payload、加密密钥和header中的加密算法
      token = JWT.encode payload, Rails.application.credentials.hmac_secret, 'HS256' 
      render json: { jwt: token }, status: :ok
    end
  ```

  19、快速完成 `Tags相关接口`
  (1) 创建Tags的model，具备两个`必传`属性：name 和 sign， 有一个`引用属性user`，和一个普通属性`deleted_at`用于标签删除
  ```rb
    bin/rails g model tag user:references name:string sign:string deleted_at:datetime
  ```

  (2) 创建Tags的controller
  ```rb
    bin/rails g controller api/v1/tags_controller
  ```


