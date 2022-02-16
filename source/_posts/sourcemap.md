---
title: Source Map
date: 2022-02-16 18:13:27
tags: Webpack
---

#### 为什么需要 Source map？

这个要从源码转换讲起，JavaScript 脚本正变得越来越复杂。大部分源码（尤其是各种函数库和框架）都要经过转换，才能投入生产环境。

> 常见的源码转换，主要是以下三种情况：
>
> （1）压缩，减小体积。比如 jQuery 1.9 的源码，压缩前是 252KB，压缩后是 32KB。
> （2）多个文件合并，减少 HTTP 请求数。
> （3）其他语言编译成 JavaScript。最常见的例子就是 CoffeeScript。

这三种情况，都使得实际运行的代码不同于开发代码，除错（debug）变得困难重重。
通常，JavaScript 的解释器会告诉你，第几行第几列代码出错。但是，这对于转换后的代码毫无用处。举例来说，jQuery 1.9 压缩后只有 3 行，每行 3 万个字符，所有内部变量都改了名字。你看着报错信息，感到毫无头绪，根本不知道它所对应的原始位置。

这就是 Source map 想要解决的问题。

#### 什么是 Source map？

简单说，Source map 就是一个信息文件，里面储存着位置信息。也就是说，转换后的代码的每一个位置，所对应的转换前的位置。

有了它，出错的时候，除错工具将直接显示原始代码，而不是转换后的代码。这无疑给开发者带来了很大方便。

![jQuery Source map](http://www.ruanyifeng.com/blogimg/asset/201301/bg2013012204.png)

#### 如何启用 Source map？

正如前文所提到的，只要在转换后的代码尾部，加上一行就可以了。

```bash
　//@ sourceMappingURL=/path/to/file.js.map
```

map 文件可以放在网络上，也可以放在本地文件系统。那这里就不得不提 `Webpack Devtool` 配置。

#### Source map 字段解读

```bash
{
  "version": 3,
  "file": "main.js",
  "sources": [
    "../src/main.js"
  ],
  "sourcesContent": [
    "throw new Error('error 1');\n"
  ],
  "names": [
    "Error"
  ],
  "mappings": ";;AAAA,MAAM,IAAIA,KAAJ,CAAU,SAAV,CAAN"
}
```

> `version`： sourceMap 版本，目前的版本是 3。
> `file`：转换后的文件名。
> `sources`：转换前的文件。该项是一个数组，表示可能存在多个文件合并。
> `sourcesContent`：source 中文件对应的源代码，是个数组
> `names`：转换前的所有变量名和属性名。
> `mappings`：记录位置信息的字符串。
>
> > 关键就是 `map` 文件的 `mappings` 属性。这是一个很长的字符串，它分成三层
> >
> > 1. 第一层是行对应，以分号 `;` 表示，每个分号对应转换后源码的一行。所以，第一个分号前的内容，就对应源码的第一行，以此类推。
> > 2. 第二层是位置对应，以逗号 `,`表示，每个逗号对应转换后源码的一个位置。所以，第一个逗号前的内容，就对应该行源码的第一个位置，以此类推。
> > 3. 第三层是位置转换，以 [VLQ](https://en.wikipedia.org/wiki/Variable-length_quantity) 编码表示，代表该位置对应的转换前的源码位置。

#### Webpack Devtool 配置

|               模式               | 解释                                                                                                         | 缺点                                                                                              |
| :------------------------------: | :----------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------ |
|              `eval`              | 每个 module 会封装到 eval 里包裹起来执行，并且会在末尾追加注释 //@ sourceURL                                 | 映射到转换后的代码，而不是映射到原始代码                                                          |
|           `source-map`           | 整个 `source map` 作为一个单独的文件生成。它为 `bundle` 添加了一个引用注释，以便开发工具知道在哪里可以找到它 | 不建议生产环境（应该将你的服务器配置为，不允许普通用户访问 `source map` 文件）                    |
|       `hidden-source-map`        | 与 `source-map` 相同，但不会为 `bundle` 添加引用注释                                                         | 不建议生产环境（不应将 `source map` 文件部署到 web 服务器。而是只将其用于错误报告工具）           |
|      `nosources-source-map`      | 创建的 `source map` 不包含 `sourcesContent`(源代码内容)                                                      | 可将 `source map` 文件部署到 web 服务器，但仍然会暴露反编译后的文件名和结构，但它不会暴露原始代码 |
|       `inline-source-map`        | `source map` 转换为 `DataUrl` 后添加到 `bundle` 中                                                           | `bundle` 体积巨增                                                                                 |
|        `cheap-source-map`        | 没有列映射(`column mapping`)的 `source map`                                                                  | 忽略 `loader source map`                                                                          |
|    `inline-cheap-source-map`     | 类似 `cheap-source-map`，但是 `source map` 转换为 `DataUrl` 后添加到 `bundle` 中                             | `bundle` 体积增大                                                                                 |
|    `cheap-module-source-map`     | 没有列映射(`column mapping`)的 `source map`，将 `loader source map` 简化为每行一个映射(mapping)              | --                                                                                                |
| `inline-cheap-module-source-map` | 类似 `cheap-module-source-map`，但是 `source map` 转换为 `DataUrl` 添加到 `bundle` 中                        | --                                                                                                |

[Webpack Devtool 配置](https://webpack.docschina.org/configuration/devtool)控制是否生成，以及如何生成 source map

#### 参考

[Webpack 官方文档 - Devtool 配置](https://webpack.docschina.org/configuration/devtool)
[聊个 5 毛钱的 Source Map](https://zhuanlan.zhihu.com/p/135228801)
[如何在正式环境无害的使用 sourceMap？](https://juejin.cn/post/6956047151843508232)
[JavaScript Source Map 详解](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)
