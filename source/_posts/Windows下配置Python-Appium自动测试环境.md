---
title: Windows下配置Python+Appium自动测试环境
date: 2021-01-16 15:25:42
tags: 技术
---


最近由于项目需要编写自动化测试脚本，就写一篇文章讲讲配环境的事情吧。


## 需要安装的东西

1. [node.JS](https://nodejs.org/en/)
2. [Android Studio](https://developer.android.com/studio)
3. [Appium](http://appium.io/)
4. [Anaconda](https://www.anaconda.com/products/individual)
5. [PyCharm](https://www.jetbrains.com/pycharm/)



## 配置

### 以下命令以管理员打开Powershell执行

1. 运行如下命令安装appium：`npm install -g appium`

2. 运行如下命令创建名为mt的虚拟环境：`conda create -n mt python=3.8`

3. 运行如下命令激活名为mt的虚拟环境：`conda activate mt`

4. 安装Python的appium库：`pip install appium-python-client`

### 设置环境变量

1. 打开Android Stuido，创建一个空工程
2. 左上角File->Project Structure->SDK Location
3. 记下Android SDK Location路径
4. 在系统变量中添加名为ANDROID_HOME的变量，值为上一步记录下的路径
5. 编辑系统变量中的Path变量，添加`%ANDROID_HOME%\tools`与`%ANDROID_HOME%\platform-tools`

### 测试

做完以上步骤后重启电脑并打开Powershell，并测试环境是否配置完成：

1. 打开安卓手机进入开发者模式，打开USB调试
2. 通过数据线将手机连接上电脑
3. 输入`adb devices`查看是否正常
4. 打开Appium
5. 点击Start Server
6. 点击右上角的放大镜
7. 在Desired Capabilities中添加一行，Name为`platformName`，Value为`Android`
8. 点击右下角的Start Session并等待
9. 若屏幕上出现手机界面则证明环境配置成功




