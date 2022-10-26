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
