---
title: 解决在Podfile中声明use_frameworks!导致部分第三方库编译失败的问题
date: 2021-03-28 15:51:11
tags: 技术
---


最近注意力全在React Native上，为了实现需求引入了许多第三方库，其中部分库出现了一些很奇怪的编译错误，例如找不到文件，链接错误等等。经过反复试错发现是由于使用了`use_frameworks!`的动态链接编译方式，某些第三方库可能不支持。在Podfile中加入以下代码声明特定库的特定编译方式即可解决。

```
pre_install do |installer|
    installer.pod_targets.each do |pod|
      if pod.name.eql?('<库的名字>')
        def pod.build_type
          Pod::BuildType.static_library
        end
      end
    end
  end
```

