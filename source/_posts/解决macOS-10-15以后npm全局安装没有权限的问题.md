---
title: 解决macOS 10.15以后npm全局安装没有权限的问题
date: 2021-03-13 18:18:33
tags: 技术
---

macOS 严格的权限控制。。。

<!-- more -->

macOS在**10.15**之后实装了更加严格的权限控制，npm全局安装的默认路径似乎已经不允许再被写入了，npm全局安装时会报红字提示没有权限，因此需要将默认的位置修改到用户的HOME下面。参考了一下StackOverflow的帖子，提供如下解决方法。

1. 打开终端。
2. 执行`npm config set prefix '~/.npm-global'`。
3. 修改终端配置文件，我的是`~/.zshrc`。
4. 添加一行`export PATH=~/.npm-global/bin:$PATH`。
5. 使配置文件生效`source ~/.zshrc`。
6. 再次尝试`npm install -g <package-name>`，问题解决。

