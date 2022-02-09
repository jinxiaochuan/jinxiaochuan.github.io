---
title: Monorepo
date: 2022-02-09 23:18:46
tags: Monorepo
---

#### 什么是 Monorepo?

Monorepo 其实不是一个新的概念，在软件工程领域，它已经有着十多年的历史了。概念上很好理解，就是把多个项目放在一个仓库里面，相对立的是传统的 MultiRepo 模式，即每个项目对应一个单独的仓库来分散管理。

Monorepo 是一种将多个项目代码存储在一个仓库里的软件开发策略（"mono" 来源于希腊语 μόνος 意味单个的，而 "repo"，显而易见地，是 repository 的缩写）。

![Monorepo-MultiRepo](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75a56317bdf94794a8b29f6cd184c888~tplv-k3u1fbpfcp-watermark.awebp)

现代的前端工程已经越来越离不开 Monorepo 了，无论是业务代码还是工具库，越来越多的项目已经采用 Monorepo 的方式来进行开发。Google 宁愿把所有的代码都放在一个 Monorepo 工程下面，Vue 3、Yarn、Npm7 等等知名开源项目的源码也是采用 Monorepo 的方式来进行管理的。

一般 Monorepo 的目录如下所示，在 packages 存放多个子项目，并且每个子项目都有自己的 package.json:

```bash
├── packages
|   ├── pkg1
|   |   ├── package.json
|   ├── pkg2
|   |   ├── package.json
├── package.json

```

那 Monorepo 究竟有什么魔力，让大家如此推崇，落地如此之广呢？

#### Monorepo 最佳实践之 Yarn Workspaces

Yarn Workspaces（工作空间/工作区，本文使用工作空间这一名称）是 Yarn 提供的 Monorepo 依赖管理机制，从 Yarn 1.0 开始默认支持，用于在代码仓库的根目录下管理多个 project 的依赖。

Yarn Workspaces 的目标是令使用 [Monorepo](https://yarnpkg.com/advanced/lexicon/#monorepository) 变得简单，以一种更具声明性的方式处理 **yarn link** 的主要使用场景。简而言之，它们允许多个项目共存在同一个代码库中，并相互交叉引用，并且保证一个项目源代码的任何修改都会立即应用到其他项目中。

重复安装、管理繁琐的缺点从 npm package 诞生起便一直存在，[node_modules hell](https://tsh.io/blog/reduce-node-modules-for-better-performance/) 就是该问题的集中体现。

![node_modules-hell](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebbecb3fc93b4276b988de0e005ab208~tplv-k3u1fbpfcp-watermark.awebp)

为了简化流程，很多大型项目采用了 Monorepo 的做法，即把所有的包放在一个仓库中管理

> Babel、React、Vue、Jest 等都使用了 monorepo 的管理方式。

Menorepo 的优点是可以在一个仓库里维护多个 package，可统一构建，跨 package 调试、依赖管理、版本发布都十分方便，搭配工具还能统一生成 CHANGELOG；
代价是即使只开发其中一个 package 也需要安装整个项目的依赖。以 jest 为例，其 Monorepo 代码结构为：

```bash
| jest/
| ---- package.json
| ---- packages/
| -------- babel-jest/
| ------------ package.json
| -------- babel-plugin-jest-hoist/
| ------------ package.json
| -------- babel-preset-jest/
| ------------ package.json
| -------- .../

```

##### 为何使用 Yarn Workspaces

在以 Monorepo 为代码组织方式的项目中，依赖管理的规模和复杂度均有不小的提升（这也不难理解，随着”数量“的增加，任何小的问题都会变得复杂）。
如何减少依赖重复安装？如何优雅实现跨目录代码共享？如何对依赖版本进行统一管理以避免版本冲突？
所以这些问题都可以借助 Yarn Workspaces 来解决！
Yarn 官方对于 Yarn Workspaces 的使用时机（Why would you want to do this?）是这样描述的：

> Your dependencies can be linked together, which means that your workspaces can depend on one another while always using the most up-to-date code available. This is also a better mechanism than yarn link since it only affects your workspace tree rather than your whole system.
>
> 工作区内的依赖关系可以链接在一起，这意味着工作区可以相互依赖，同时始终使用最新的可用代码。这也是一个相对于 yarn link 更好的机制，因为它只影响你的工作空间树，而不是整个系统。
>
> All your project dependencies will be installed together, giving Yarn more latitude to better optimize them.
>
> 所有的项目依赖关系都将被安装在一起，为 Yarn 提供更多的自由度来更好地优化它们。
>
> Yarn will use a single lockfile rather than a different one for each project, which means fewer conflicts and easier reviews.
>
> 对于每个项目，Yarn 将使用一个公共的的锁文件而不是为每个工程使用一个不同的锁文件，这意味着更少的冲突和更容易的版本审查。

##### 如何启用 Workspace

> 1. 确保项目中安装有 yarn
> 2. 在项目根目录的 packag.json 中增加如下配置:
>
> ```bash
>   {
>     "private": true,
>     "workspaces": ["packages/*"]
>   }
> ```

#### monorepo 方案实践

##### 锁定环境：Volta

[Volta](https://volta.sh/) 是一个 JavaScript 工具管理器，它可以让我们轻松地在项目中锁定 node，npm 和 yarn 的版本。你只需在安装完 Volta 后，在项目的根目录中执行 volta pin 命令，那么无论您当前使用的 node 或 npm（yarn）版本是什么，volta 都会自动切换为您指定的版本。

相较于 nvm，Volta 还具有一个诱人的特性：当您项目的 CLI 工具与全局 CLI 工具不一致时，Volta 可以做到在项目根目录下自动识别，切换到项目指定的版本

##### 复用 packages：workspace

使用 monorepo 策略后，收益最大的两点是：

避免重复安装包，因此减少了磁盘空间的占用，并降低了构建时间；
内部代码可以彼此相互引用；
这两项好处全部都可以由一个成熟的包管理工具来完成，对前端开发而言，即是 yarn（1.0 以上）或 npm（7.0 以上）通过名为 workspaces 的特性实现的（⚠️ 注意，支持 workspaces 特性的 npm 目前依旧不是 LTS 版本）。

为了实现前面提到的两点收益，您需要在代码中做三件事：

1. 调整目录结构，将相互关联的项目放置在同一个目录，推荐命名为 packages；
2. 在项目根目录里的 package.json 文件中，设置 workspaces 属性，属性值为之前创建的目录；
3. 同样，在 package.json 文件中，设置 private 属性为 true（为了避免我们误操作将仓库发布）；

经过修改，您的项目目录看起来应该是这样：

```bash
.
├── package.json
└── packages/
    ├── @mono/project_1/ # 推荐使用 `@<项目名>/<子项目名>` 的方式命名
    │   ├── index.js
    │   └── package.json
    └── @mono/project_2/
        ├── index.js
        └── package.json

```

而当您在项目根目录中执行 npm install 或 yarn install 后，您会发现在项目根目录中出现了 node_modules 目录，并且该目录不仅拥有所有子项目共用的 npm 包，还包含了我们的子项目。因此，我们可以在子项目中通过各种模块引入机制，像引入一般的 npm 模块一样引入其他子项目的代码。
请注意我们对子项目的命名，统一以 @<repo_name>/ 开头，这是一种社区最佳实践，不仅可以让用户更容易了解整个应用的架构，也方便您在项目中更快捷的找到所需的子项目。
至此，我们已经完成了 monorepo 策略的核心部分，实在是很容易不是吗？但是老话说「行百里者半九十」，距离优雅的搭建一个 monorepo 项目，我们还有一些路要走。

##### 统一配置：合并同类项 - Eslint，Typescript 与 Babel

###### TypeScript

我们可以在 packages 目录中放置 tsconfig.settting.json 文件，并在文件中定义通用的 ts 配置，然后，在每个子项目中，我们可以通过 extends 属性，引入通用配置，并设置 compilerOptions.composite 的值为 true，理想情况下，子项目中的 tsconfig 文件应该仅包含下述内容：

```bash
{
  "extends": "../tsconfig.setting.json", // 继承 packages 目录下通用配置
  "compilerOptions": {
    "composite": true, // 用于帮助 TypeScript 快速确定引用工程的输出文件位置
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}

```

###### Eslint

对于 Eslint 配置文件，我们也可以如法炮制，这样定义子项目的 .eslintrc 文件内容：

```bash
{
  "extends": "../../.eslintrc", // 注意这里的不同
  "parserOptions": {
    "project": "tsconfig.json"
  }
}
```

###### Babel

Babel 配置文件合并的方式与 TypeScript 如出一辙，甚至更加简单，我们只需在子项目中的 .babelrc 文件中这样声明即可：

```bash
{
  "extends": "../.babelrc"
}
```

当一切准备就绪后，我们的项目目录应该大致呈如下所示的结构：

```bash
.
├── package.json
├── .eslintrc
└── packages/
    │   ├── tsconfig.settings.json
    │   ├── .babelrc
    ├── @mono/project_1/
    │   ├── index.js
    │   ├── .eslintrc
    │   ├── .babelrc
    │   ├── tsconfig.json
    │   └── package.json
    └───@mono/project_2/
        ├── index.js
        ├── .eslintrc
        ├── .babelrc
        ├── tsconfig.json
        └── package.json

```

##### 统一命令脚本：[scripty](https://www.npmjs.com/package/scripty)

如果您的子项目足够多，您可能会发现，每个 package.json 文件中的 scripts 属性都大同小异，并且一些 scripts 充斥着各种 Linux 语法，例如管道操作符，重定向或目录生成。重复带来低效，复杂则使人难以理解，这都是需要我们解决的问题。

这里给出的解决方案是，使用 scripty 管理您的脚本命令，简单来说，scripty 允许您将脚本命令定义在文件中，并在 package.json 文件中直接通过文件名来引用。这使我们可以实现如下目的：

子项目间复用脚本命令；
像写代码一样编写脚本命令，无论它有多复杂，而在调用时，像调用函数一样调用；

通过使用 scripty 管理我们的 monorepo 应用，目录结构看起来将会是这样：

```bash
.
├── package.json
├── .eslintrc
├── scirpts/ # 这里存放所有的脚本
│   │   ├── packages/ # 包级别脚本
│   │   │   ├── build.sh
│   │   │   └── test.sh
│   └───└── workspaces/ # 全局脚本
│           ├── build.sh
│           └── test.sh
└── packages/
    │   ├── tsconfig.settings.json
    │   ├── .babelrc
    ├── @mono/project_1/
    │   ├── index.js
    │   ├── .eslintrc
    │   ├── .babelrc
    │   ├── tsconfig.json
    │   └── package.json
    └── @mono/project_2/
        ├── index.js
        ├── .eslintrc
        ├── .babelrc
        ├── tsconfig.json
        └── package.json
```

注意，我们脚本分为两类「package 级别」与「workspace 级别」，并且分别放在两个文件夹内。这样做的好处在于，我们既可以在项目根目录执行全局脚本，也可以针对单个项目执行特定的脚本。

通过使用 scripty，子项目的 package.json 文件中的 scripts 属性将变得非常精简：

```bash
{
  ...
  "scripts": {
    "test": "scripty",
    "lint": "scripty",
    "build": "scripty"
  },
  "scripty": {
    "path": "../../scripts/packages" // 注意这里我们指定了 scripty 的路径
  },
  ...
}
```

##### 统一包管理：[Lerna](https://www.lernajs.cn/)

![lerna](https://www.lernajs.cn/images/lerna-hero.svg)

> 当多个子项目放在一个代码仓库，并且子项目之间又相互依赖时，我们面临的棘手问题有两个：
>
> 1. 如果我们需要在多个子目录执行相同的命令，我们需要手动进入各个目录，并执行命令；
>
> 2. 当一个子项目更新后，我们只能手动追踪依赖该项目的其他子项目，并升级其版本。

通过使用 Lerna，这些棘手的问题都将不复存在。
当在项目根目录使用 `npx lerna init` 初始化后，我们的根目录会新增一个 `lerna.json` 文件，默认内容为：

```bash
{
  "packages": ["packages/*"],
  "version": "0.0.0"
}
```

让我们稍稍改动这个文件，使其变为：

```bash
{
  "packages": ["packages/*"],
  "npmClient": "yarn",
  "version": "independent",
  "useWorkspaces": true,
}
```

可以注意到，我们显示声明了我们的包客户端（`npmClient`）为 `yarn`，并且让 Lerna 追踪我们 `workspaces` 设置的目录，这样我们就依旧保留了之前 workspaces 的所有特性（子项目引用和通用包提升）。

除此之外一个有趣的改动在于我们将 `version` 属性指定为一个关键字 `independent`，这将告诉 lerna 应该将每个子项目的版本号看作是相互独立的。当某个子项目代码更新后，运行 `lerna publish` 时，Lerna 将监听到代码变化的子项目并以交互式 CLI 方式让开发者决定需要升级的版本号，关联的子项目版本号不会自动升级，反之，当我们填入固定的版本号时，则任一子项目的代码变动，都会导致所有子项目的版本号基于当前指定的版本号升级。

Lerna 提供了很多 CLI 命令以满足我们的各种需求，但根据 2/8 法则，您应该首先关注以下这些命令：

- `lerna init`：常见一个新的 lerna 仓库（repo）或将现有的仓库升级为适配当前 版本的 Lerna。
  参数 `--independent/-i` – 使用独立的版本控制模式
- `lerna bootstrap`：等同于 `lerna link` + `yarn install`，用于创建符合链接并安装依赖包；
- `lerna run`：会像执行一个 for 循环一样，在所有子项目中执行 npm script 脚本，并且，它会非常智能的识别依赖关系，并从根依赖开始执行命令；
- `lerna exec`：像 `lerna run` 一样，会按照依赖顺序执行命令，不同的是，它可以执行任何命令，例如 shell 脚本；
- `lerna publish`：发布代码有变动的 package，因此首先您需要在使用 Lerna 前使用 git commit 命令提交代码，好让 Lerna 有一个 baseline；
- `lerna add`：将本地或远程的包作为依赖添加至当前的 monorepo 仓库中，该命令让 Lerna 可以识别并追踪包之间的依赖关系，因此非常重要；

#### 参考

[All in one：项目级 monorepo 策略最佳实践](https://juejin.cn/post/6924854598268108807)
[Monorepo 最佳实践之 Yarn Workspaces](https://juejin.cn/post/7011024137707585544)
[现代前端工程为什么越来越离不开 Monorepo?](https://juejin.cn/post/6944877410827370504)
[lerna 多包管理实践](https://juejin.cn/post/6844904194999058440)
