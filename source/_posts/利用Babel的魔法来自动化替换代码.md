---
title: 利用Babel的魔法来自动化替换代码
date: 2024-06-02 11:24:08
tags: 技术
---

这期我们来讲讲能用babel的魔法干啥。。。

<!-- more -->


上大学的时候有一门计算机基础课叫编译原理，当时学的时候一直一头包，心里想的是以后工作的时候应该是用不到里面的知识的，毕竟我觉得以我的水平应该也到不了能写编译器的程度。

但是没想到的是工作了几年，阴差阳错地干起了大前端方向的事情，更是没想到编译这件事情居然成为了现在大前端圈子里面绕不过去的话题。babel更是大前端编译这件事情的重中之重。结合最近工作中对它的应用来简单地聊聊吧。


事情是这样子的，随着项目的迭代与重构，我们用到的一个模块本来是从a导出的，现在要改到从b导出，示例如下：

```ts
import { module } from 'a' 
// 改成下面这样
import { module } from 'b'
```

搜了一下代码库发现全局有大概600多处地方，一个个改肯定是要改麻的。于是乎肯定是要想一些自动化的办法。一开始想的是编辑器全局替换，但是不知道怎么处理下面这种情况：

```ts
import { module, module1 } from 'a'
// 改成下面这样
import { module1 } from 'a'
import { module } from 'b'
```
于是作罢，那思路就来到了写一个脚本来做这个事情，先试了下正则匹配，发现也不太好搞，比如上述的这种情况要怎么处理呢。尤其是代码是格式化过的，有时候是长这样的：
```ts
import { 
    module,
    module1,
    module2,
    module3,
} from 'a'
```
用正则处理起来就很麻烦，于是只能用终极解决方案：babel。

关于babel与AST抽象语法树的具体故事我这里就不多做赘述了，网上已经有很多优秀的文章了，笔者这里只单纯地讨论如何利用它来进行工程化的实现。


说白了，我们要实现的其实就是一个解析TS/TSX文件并替换导入导出的脚本。

要实现替换，那么第一步就是要遍历我们的代码库下面的所有文件并拿到文件的内容，这步相当简单，利用Node.JS提供的`fs`库即可：

```js
// 递归遍历目录下的所有文件
const fs = require('fs')
const path = require('path')

function traverseDirectory(dir) {
  const files = fs.readdirSync(dir)
  files.forEach(file => {
    const filePath = path.join(dir, file)
    const stats = fs.statSync(filePath)
      // 如果是目录，则继续递归遍历
    if (stats.isDirectory()) {
      traverseDirectory(filePath)
    } else if (
        // 只处理js，ts，tsx文件
      stats.isFile() &&
      (path.extname(file) === '.js' ||
        path.extname(file) === '.ts' ||
        path.extname(file) === '.tsx')
    ) {
        // 具体对于文件操作的逻辑在这里执行
        // transformFile(filePath)
    }
  })
}
```

我们现在已经能够拿到目录下面的所有代码文件了，那么下一步就是实现`transformFile`方法做转换，这里就要用到`@babel/core`提供的`transformFileSync`方法了，这个方法读取文件内容，应用我们配置的预设和插件，最终返回转换后的代码与抽象语法树。

那么函数的大概结构就是这样：

```js
// result就是结果，包含我们转换完成的代码和对应的AST
const result = transformFileSync(filePath, {
    // 要实现我们的需求目前不需要任何预设
    presets: [],
    // 这里配置插件
    plugins: [
      [
          // 需要使用这个插件来赋予babel解析TS/TSX代码的能力
        'module:@babel/plugin-syntax-typescript',
          // 该插件相关的配置，根据要解析的代码略有区别，具体可以查阅官方文档
        {
          disallowAmbiguousJSXLike: false,
          isTSX: true
        }
      ],
        // 这里就是我们自己要写的插件
      ({ types: t }) => {
        return {
          visitor: {
              // 实现转换的核心逻辑
          }
        }
      }
    ]
  })
```

`visitor`是babel转换的核心逻辑所在，开发者可以通过实现里面具体的转换函数来实现想要的逻辑。对于我们这个场景来说，我们需要处理的文件顶部的导入语法的内容，那么就可以这么写：
```js
({ types: t })=>{
    visitor: {
        // Program是babel解析的AST的最顶层的节点，代表了文件的整棵AST
        Program(path) {
            
        }
    }
}
```

这个时候可能有点看不明白是啥意思，我把代码和对应的AST放出来大家就懂了：
```ts
import { module } from 'a'
import { module1 } from 'b'
```

对于上述代码，Program所拿到的对象就是这么个玩意，大家可以用[AST Explorer](https://astexplorer.net/)来自己玩玩：
```json
{
  "type": "Program",
  "start": 0,
  "end": 54,
  "body": [
    {
      "type": "ImportDeclaration",
      "start": 0,
      "end": 26,
      "specifiers": [
        {
          "type": "ImportSpecifier",
          "start": 9,
          "end": 15,
          "imported": {
            "type": "Identifier",
            "start": 9,
            "end": 15,
            "name": "module"
          },
          "local": {
            "type": "Identifier",
            "start": 9,
            "end": 15,
            "name": "module"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "start": 23,
        "end": 26,
        "value": "a",
        "raw": "'a'"
      }
    },
    {
      "type": "ImportDeclaration",
      "start": 27,
      "end": 54,
      "specifiers": [
        {
          "type": "ImportSpecifier",
          "start": 36,
          "end": 43,
          "imported": {
            "type": "Identifier",
            "start": 36,
            "end": 43,
            "name": "module1"
          },
          "local": {
            "type": "Identifier",
            "start": 36,
            "end": 43,
            "name": "module1"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "start": 51,
        "end": 54,
        "value": "b",
        "raw": "'b'"
      }
    }
  ],
  "sourceType": "module"
}
```
可以看到已经拿到我们所需要的信息了，在babel所生成的AST里面。整个`import { module } from 'a'`是一个叫做`ImportDeclaration`的对象，里面的`module`是一个叫做`ImportSpecifier`的对象，来源`a`是一个叫做`Literal`的玩意。那么脚本的思路就从操作字符串变成了操作这棵树，只要把`ImportDeclaration`和里面的`ImportSpecifier`以及`Literal`等节点变成我们想要的就可以，那首先我们先找到导入源为a的节点，做一下处理：

```js
({ types: t })=>{
    visitor: {
        // Program是babel解析的AST的最顶层的节点，代表了文件的整棵AST
        Program(path) {
            const specifiersToMove = []
            body.forEach((node, index) => {
                // 找到从a导入模块的代码
                if (
                    node.type === 'ImportDeclaration' &&
                    node.source.value === 'a'
                ) {
                    const newSpecifiers = []
                    // 遍历ImportSpecifier，如果是a的话就删掉
                    node.specifiers.forEach(specifier => {
                        if (
                            !['a'].includes(specifier.local.name)
                        ) {
                            newSpecifiers.push(specifier)
                        } else {
                            // 一会要挪位置的ImportSpecifier
                            specifiersToMove.push(specifier)
                        } 
                    })
                    node.specifiers = newSpecifiers
                }
            })
        }
    }
}
```

执行上述代码后，如果这时候生成代码的话就是这个效果：
```ts
import {} from 'a'
import { module1 } from 'b'
```

可以看到`module`已经被删除了，但是还保留着引入这一行，其实不太对，应该把整行都删掉才对，这里先不管，我们先去操作下一行把a写到`b`的引入里面：
```js
body.forEach(node => {
    // 和找a一样找到b
    if (
      node.type === 'ImportDeclaration' &&
      node.source.value === 'b'
    ) {
      node.specifiers = Array.from(new Set([...node.specifiers, ...specifiersToMove]))
    }
})
```

和之前操作a的代码拼在一起执行就能把代码转换成这样：
```js
import {} from 'a'
import { module1, module } from 'b'
```

这个时候需求基本上就完成了，读到这里的观众可以思考一下怎么处理`import {} from 'a'`这种空引用的问题。说白了其实就是如果操作后变成空引用了直接把对应的整句`ImportDeclaration`给铲了就行：
```js
// hasAImport和aIndex可以在遍历的时候暂存下来，就是找到的'a'的specifier的位置
if (!hasAImport && aIndex !== -1) {
    body.splice(aIndex, 1)
}
```

这样代码就被转换成了这样, 空引用就消失了：
```js
import { module1, module } from 'b'
```

接下来还要处理一种情况，就是代码长这样：

```js
import { module } from 'a'
```

这个时候从b引用的这句话是没有的，怎么办，那这个时候就需要继续利用babel的能力去构造一个出来:

```js
if (!hasBImport) {
    const tempBody = [...body]
    // 构造一个ImportDeclaration并在原来的a import语句那行下面插入
    tempBody.splice(
      aIndex,
      0,
      t.importDeclaration(
        specifiersToMove,
        t.stringLiteral('b')
      )
    )
    path.node.body = tempBody
}
```
跑完就变成了这样：
```ts
import { module } from 'b'
```

至此需求基本完成，最后覆盖原来的文件就可以：

```js
fs.writeFileSync(filePath, result.code)
```

其实有了这个思路之后做前端工程化的时候很多重构可以用babel去完成，就像本文所说的导入更新。一些实际做项目交付时的代码脱敏也可以用这个来搞


