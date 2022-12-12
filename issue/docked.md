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