---
title: Monorepo怎么热更新？
date: 2024-06-29 10:47:25
tags: 技术
---

一个搭建Monorepo的时候都会遇到的问题

<!-- more -->

这些年的前端项目基本都是Monorepo了，一个Monorepo里面往往有多个子包。像这样:
```
- packages
    - A
    - B
    - C
    ...
- package.json
```

在整个Monorepo的开发维护过程中，有这样一个场景：开发在开发A包，但是同时需要对B包的代码做修改，对于B包的修改能否及时反馈在A的页面上就是很重要的事情了，毕竟谁都不希望改完B包还要手动构建一把。

一般一个包的package.json里面会有这样的信息：
```json
{
  "exports": {
    ".": {
      "import": "./dist/package.js"
    }
  }
}
```
以上这段配置表示当别的地方通过ESM风格的`import { x } from 'A'`引用A的代码时，解析器会把A的根目录下的`dist/package.js`作为入口文件去找所引用的东西。这就是为啥改了B之后还需要构建一把才能在A的页面看到效果，因为对于A来说，它引用的是产物，而非B的源代码，只有B的产物更新了，才会触发A的热更新。

那么很容易想到，那我不让A引用B的构建产物，而是直接引用B的源代码不就行了吗？我们对A的`vite.config.ts`做如下配置：

```ts
export default defineConfig({
    resolve: {
        alias: {
            // 直接把B的引用手动重定向到源码的index.ts入口文件
            "B": path.resolve(__dirname, "../b/src/index.ts")
        }
    }
})
```

试一下，热更新正常，但是似乎有点麻烦，以后不是每加一个包都需要在用到的地方再手动配一个alias？其实有一种更优雅的办法，我们回到B的package.json做如下修改:
```json
{
    "exports": {
        ".": {
            "import": {
              "development": "./src",
              "default": "./dist/package.js"
            }
        }
    }
}
```
可以看到我们在`import`里面分别配置了`development`和`default`，核心就是这句`development`的配置，意思就是告诉解析器在dev环境下使用`./src`目录下面的代码，即B包的源码。