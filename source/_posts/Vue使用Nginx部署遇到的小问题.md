---
title: Vue使用Nginx部署遇到的小问题
date: 2021-01-14 01:29:17
tags:
---


# Vue使用Nginx部署遇到的小问题

今天兴冲冲地把自己博文的链接分享给同事看，结果同事打开链接给我发了下面这张图：

![](https://minrit-1255311621.cos.ap-shanghai.myqcloud.com/blog_resource/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20210114012437.jpg)

看到的一瞬间麻了，然后想了想，应该是nginx没有对url进行正确的路由查询，谷歌之后在nginx配置文件添加如下配置解决问题：

![](https://minrit-1255311621.cos.ap-shanghai.myqcloud.com/blog_resource/QQ%E6%88%AA%E5%9B%BE20210114012241.png)
