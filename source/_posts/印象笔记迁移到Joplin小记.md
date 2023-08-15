---
title: 印象笔记迁移到Joplin小记
date: 2023-08-15 13:05:12
tags: 
    - 技术 
    - 日常
---


印象笔记最近广告越来越频繁了，用起来也不是很省心，而且印象笔记很多高阶功能我都用不到，我只需要单纯的记笔记和云同步两个功能，为了这每年付一笔钱不是很值。思考了一下选择拥抱开源--Joplin。

<!-- more -->

## 导出印象里面的笔记

本来以为点 Export 按钮就可以轻松的吧笔记弄出来了，结果发现现在版本的印象笔记没导出ENEX 格式的选项了。这摆明了是要圈养用户，更要赶紧跑了。谷歌了一下发现有个开源库可以把印象笔记同步下来，叫[Evernote Backup](https://github.com/vzhd1701/evernote-backup)。

1. 安装`brew install evernote-backup`
2. 登录`evernote-backup init-db --user <username> --password <password> --backend china`
3. 同步`evernote-backup sync`
4. 导出`evernote-backup export <path>`

简简单单几步就把 ENEX 格式的笔记导出了，还是开源社区给力啊。。

## 导入Joplin

这个就没门槛了，直接导入就行了，Joplin 有提供两个选项，一个是 md 格式的 enex，另一个是 html 格式的。笔者两个都试了，感觉 html 的格式会不对，于是用的是 md 格式的导入方式。


## 同步

Joplin 有提供多种同步方式，有官方的 Joplin Cloud，也有 WebDav，S3 等方式。虽说笔者有白群晖，但是这里还是选择了 S3的方式，主要是考虑到之前用白群晖的内网穿透的时候，发现速度比较慢。又因为笔者一直用的腾讯云的 COS，于是乎就直接在腾讯云上面建了一个存储桶。

1. 创建存储桶
2. 创建用户给 Joplin 用
3. 给用户授权
4. 配置 Joplin
5. 同步


通过上面五步就可以完成 Joplin 的云端同步了。又减少了一笔不用的开支，开心～
