---
title: Jenkins做个博客的CICD
date: 2023-08-13 11:45:03
tags: 技术
---

HEXO 通过 Jenkins 部署小记

<!-- more -->


HEXO 是一个很好用的博客框架，可以轻松通过命令的方式生成新的博客文章，撰写完毕后打包部署就好。我个人比较懒，期望的是每件事情能自动化，于是用了经典的 CICD 工具来做个部署


## 安装 Jenkins

笔者使用的是 macOS，直接运行如下命令即可：

```
brew install jenkins-lts
```

安装完成后运行如下命令启动 Jenkins：

```
brew services start jenkins-lts
```

然后在浏览器中打开 `http://localhost:8080` 即可看到 Jenkins 的首页，照着页面提示安装一些插件做一些配置即可。


## 配置流水线

因为博客本身会推送到 gitee 或者是 github 的库上面，然后博客本身是个 ECS，所以思路其实就是让 Jenkins 通过 SSH 的方式登录到 ECS 上面，然后执行一些命令来完成部署。


### 首先先准备好一个在云端跑的SHELL脚本

```
cd ~/new_blog/<my-blog>/
git pull
hexo generate
rm -rf /etc/nginx/hexo/public
mv /root/new_blog/<my-blog>/public /etc/nginx/hexo/
nginx -s reload
```

其实就很简单，就是拉取最新的博文，然后生成静态文件产物，随后将产物移到配置好的目录下面，然后重启 nginx 即可。


### Jenkins 配置

1. 安装一个叫`SSH Plugin`的插件，用于支持 SSH 的方式登录到远程机器上面。
2. 创建一个 Jenkins Credentials，用于存储 ECS 的登录信息。
3. 创建一个 Site，指向远程ECS。
4. 新建一条流水线，在构建任务里面加一个叫`Execute shell script on remote host using ssh`，输入如下命令：
    ```
   source /root/.profile
   source /root/.bashrc
   bash -xe deploy_blog.sh
   ```
   其中`deploy_blog.sh`就是上面准备好的脚本。 头两句 source命令是为了让 Jenkins 有远程 ECS 的 bash 运行环境。
5. 保存配置即可。


以上就是大概的配置流程，以后写完博客直接启动 Jenkins 的流水线大概 5～6 秒就可以完成部署。

