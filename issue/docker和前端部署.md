# 1、什么是docker?
```typescript
  Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖到一个可以抑制的容器中，然后发布到任何流行的linux机器上，也可以实现虚拟化。容器完全使用沙盒机制，相互之间话不会存在任何接口；
  是一种不依赖于语言、框架或者包装系统的虚拟化技术；
  在前后分离的项目中，我们可以开发环境在docker开发，生产环境通过docker部署。
```

# 2、什么是镜像?
```typescript
  docker的镜像是一种轻量级的、可执行的独立软件包。用来打包软件运行环境和基于运行环境的开发软件，它包含运行某个软件所需要的代码、运行时、库、环境变量和配置文件等。
```

# 3、docker加速?
```typescript
  1、阿里云申请个人容器镜像用于docker加速
  2、进入镜像工具-镜像加速器，将加速期地址添加到Docker Engine配置项如：
    "registry-mirrors": [
      "https://rxvtxxcz.mirror.aliyuncs.com"
    ]
```

**常用命令：**
```typescript
1、docker version // 查看docker版本
2、docker search // 在hub中搜索和docker相关信息如查找centos镜像
3、docker pull // 拉取镜像
4、docker push // 推送镜像到hub
4、docker info // 查看docker信息，例如在Registry Mirrors中可以看到用于docker加速的阿里云源地址
5、docker images // 查看已经下载的镜像
```

# 4、docker安装
  1、mac可以通过`Homebrew`来安装Docker
  2、先安装`Homebrew`，因官网安装`brew`的源在国外，所以需要切换安装的源到国内：
  ```typescript
    /bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
  ```  
  3、用`Homebrew`安装`docker`:
  ```typescript
    brew install --cask --appdir=/Applications docker
  ```
  4、安装完成，用 `docker --version` 查看版本
  5、进入docker客户端的设置中的：`Docker Engine`选项卡，配置如下加速：
  ```json
  {
    "builder": {
      "gc": {
        "enabled": true,
        "defaultKeepStorage": "20GB"
      }
    },
    "experimental": false,
    "features": {
      "buildkit": true
    },
    "registry-mirrors":[
      "http://hub-mirror.c.163.com"
    ]
  }
  ```
  6、使用docker info验证加速配置
    

**5、工作空间：**
  1、使用Vscode插件`Dev Container`打开docker镜像，再使用命令`pwd`，可以看到当前所处的工作空间
  ```typescript
    /workspaces/oh-my-env
  ```
  2、这个工作空间就代表了一个`容器内外共享`的目录，即外部的mac系统和内部的linux系统的文件能够得到`同步`
  3、一般需要共享的文件是放在这个`共享工作空间`，常规的非共享操作可以在～/repos等其他目录中进行

**6、使用docker搭建rails+vue3+postgresql的全栈应用：**
  1、为了统一开发环境和生产环境，我使用docker来作为容器；
    （1）打开容器前，先在终端执行：docker network create network1，来创建一个独立的网络
    （2）使用Vscode插件`Dev Container`打开docker镜像
    （3）该docker镜像自带了以下功能：archlinux、zsh、fzf、rvm + ruby、nvm + node、go、
      docker in docker、chezmoi、各种国内加速……
    （4）该环境使用的核心依赖的版本分别是：`nodeJs v18.10.0`, `Rails 7.0.4`,`pnpm 7.12.2`

  2、在docker中，初始化后端项目：
  ```sh
    gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
    bundle config mirror.https://rubygems.org https://gems.ruby-china.com
    gem install rails -v 7.0.2.3
    pacman -S postgresql-libs
    cd ~/repos
    rails new --api --database=postgresql --skip-test mangosteen-1
  ```

  3、在mac环境中，初始化 & 启动 -> 和docker中rails项目相关联的数据库
  ```sh
    docker run -d \
    --name db-for-mangosteen \
    -e POSTGRES_USER=mangosteen \
    -e POSTGRES_PASSWORD=123456 \
    -e POSTGRES_DB=mangosteen_dev \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v mangosteen-data:/var/lib/postgresql/data \
    --network=network1 \
    postgres:14
  ```

  4、在docker中，修改 config中的`database.yml`文件，连接数据库
  ```yml
    development:
    <<: *default
    database: mangosteen_dev
    username: mangosteen
    password: 123456
    host: db-for-mangosteen
  ```

  5、在docker中运行rails应用，该应用会为我们默认启动一个`3000端口`的服务，在外部mac环境可以访问到；至此，初步完成docker的后端环境配置。
  ```sh
    bundle exe rails s
  ```

  6、使用rails的`基础操作`记录
  (1) 建表 - 建立`postgresql` 中的一个`model`模型
  ```sh
    # 在docker的rails应用目录 -> 首先建立user模型，具备emial和name字段
    bin/rails g model user email:string name:string
  ```
  可以发现会为我们创建一个继承于 `ActiveRecord::Migration`类的user表
  ```rb
    class CreateUsers < ActiveRecord::Migration[7.0]
      def change
        create_table :users do |t| # |t|类似于箭头函数的参数
          t.string :email
          t.string :name
          t.timestamps
        end
      end
    end
  ```
  (2) 我们还可以在表中手动添加其他的`key`，或给某些字段添加`limit`等限制;然后再执行 `db:migrate` 来同步到数据库
  ```sh
    bin/rails db:migrate
  ```
  也可以执行 `db:rollback step=1` 撤销上一次的数据库保存操作
  ```sh
    bin/rails db:rollback step=1
  ```
  (3) 创建控制器 `controller`
  ```rb
    bin/rails g controller users show create
  ```
  (4) 我们可以进入 `config/routes.rb` 来 `创建路由`
  ```rb
    get '/users/:id', to: 'users#show'
    post '/users/', to: 'users#create'
  ```

  **7、前端部署关键点：**
  ```typescript
    1、管理公司测试服务器环境，了解linux常见命令以及docker/nginx，并通过docker镜像与nginx的转发独立完成前端部署
    2、独立编写和管理前端项目的Dockerfile，并将nginx配置文件放在项目中独立管理，并独立编写ci文件（如DockerFile的ADD命令会添加内容到镜像，如果是tar的压缩包，会自动解压缩；也可以是文件、路径或链接）
      Dockerfile - ADD：添加内容到镜像
      Dockerfile - RUN：创建镜像
      Dockerfile - ENV：制定镜像的环境
      Dockerfile - CMD：Docker镜像被启动后容器默认执行的命令
      Dockerfile - ENTRYPOINT：容器启动时第一个执行的命令和参数
    3、Dockerfile优化
      - 构建缓存：如果依赖没变，则不重新安装依赖，而复用之前的，先ADD添加Gemfile、Gemfile.lick、vendor/cache, 然后创建镜像RUN bundle install --local（GEMFILE、GEMFILE.lock、vendor未变，则RUN bundle install则不会变而是使用缓存），一般将一些不常改动的文件放Dockerfile的前面
      - 多阶段构建：在Dockerfile中使用多个FROM指令，每个FROM指令可以使用不同的基镜像，且每条FROM指令会开始新阶段的构建。在多阶段构建中，我们可以将资源从一个阶段复制到另一个阶段，大幅度减少镜像体积，优化前端部署时间
      （tip小问题：镜像为什么需要上传和下载：在公司里镜像一般是需要先在一个系统构建，然后可能在另一个系统里进行部署，所以这之间的传输就设计了上传和下载）
    4、熟悉nginx和webpack的hash资源强缓存配置：
      - 什么是强缓存：浏览器不会再次向服务器发起请求而是去缓存里取资源，Cache Control设置max age；CDN上也会有一层缓存，这样拿资源可以更快
      - 为何可以对hash资源配置长期强缓存：因为资源一旦改变，打包后的hash也会发生改变，这样可以作为资源是否更新的标识，因此可以对hash资源加上长期强缓存
      - 如何对hash缓存进行强化：hash缓存有一个缺陷，即一个文件发生改变，则依赖它的chunk也会更新，一般来说可以固定模块id+固定chunkid来做到缓存优化（即一个模块引入两个chunk，其中一个变了，打包出来的另外一个chunk依然可以走缓存）
    5、nginx的try_files 适配history路由
    6、静态资源上传cdn
  ```