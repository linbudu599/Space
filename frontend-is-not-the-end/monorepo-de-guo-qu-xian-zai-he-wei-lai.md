---
description: '2022-04-23'
---

# Monorepo 的过去、现在、和未来

最近在准备 [QCon+ Monorepo 专题](https://qconplus.infoq.cn/2022/beijing/track/1325) 的分享，突然发现 Monorepo 方案其实一直在不断演进，每一个阶段都解决了上一个时代的数个核心问题。因此，这里简单地写一篇文章分享下 Monorepo 方案至今的进化路线（真的非常非常之简单）。

#### 刀耕火种：伪 Monorepo

在这一时代，lerna 等方案还未出现，除了各大公司如 Google、Meta 自建的巨型 Monorepo 方案以外，外界开发者常用的 Monorepo 方案就是将所有的项目直接放进一个文件夹/仓库下，比如这样：

```
- web-app
- mobile-app
- web-admin
- miniprogram
- node-server
- node-bff
- rpc
- ...
```

严格来说，这一组织方式并不属于 Monorepo，它只是把所有项目放在一起，每个项目自己安装一次依赖，如果项目之间存在依赖关系，需要自己手动 link 。项目的开发、发布、CI/CD 等同样会让你怀疑人生：在五个 Terminal 之间来回切换一个个启动项目、手动改版本然后按照依赖项目-主体项目的顺序发布、足够你开几把游戏的 CI/CD 时间。

使用这种方式和使用 Polyrepo，即一个项目一个仓库的管理方式并没有什么区别，因此我们这里称它是伪·Monorepo。要想成为一个真正意义上的 Monorepo，我理解至少有两点入门门槛需要满足：

* 项目之间存在自治的依赖关系。这里的自治，指的是不需要人工按照拓扑排序进行项目之间的 link，而是有一定的自动化程度。比如在上面的例子里如果我们提供一个 script/link.js 文件，那么也可以勉强认为满足了这一点。项目依赖关系的构建主要是在开发时起到作用，如你可以直接在主项目中 link 子项目，然后对子项目的修改能够立刻在主项目中产生效果。
* 工程子项目的依赖（node\_modules）之间并非是完全独立的，对于版本一致的三方包，只应该被安装一次。以及，对于对单例有要求的依赖包如 React，版本之间存在冲突又可能并存的包如 Webpack，ESBuild / SWC / NodeSass 这一类非 JavaScript 编写的依赖包，都需要得到妥善的处理。

总结一下其实就是这么两个问题：

* 项目依赖关系构建
* 工程依赖处理

实际上，这一刀耕火种阶段的 Monorepo 方案其实现在很难见到使用了，毕竟这么做还不如拆分成一个项目一个仓库的 Polyrepo 架构。很快，我们迎来了新的、面向社区的 Monorepo 方案，它们很好地解决了上面的两个问题，为我们带来了真正的 Monorepo 开发体验。

> 面向社区：Google 、Meta、Microsoft 等公司的 Monorepo 方案其实此时还未开源出来，或者说并不适合社区的普通个人开发者 / 团队来使用。

#### 工业革命：Task Runner 与 Workspace

现在，应该很少有没有听说过 lerna 的前端同学了，这里也不准备介绍它的使用方式。如果还未了解，你可以阅读官方文档。lerna 这一方案要解决的核心问题，其实正对应着上面列出的两个痛点。

对于项目依赖关系构建，lerna 在初始化项目时（lerna bootstrap）会自动地为存在依赖关系的子项目创建 link，同时也支持了依赖关系创建（lerna add）、基于依赖关系的版本升级与发布（lerna version、lerna publish）等。lerna 其实还支持 --include-dependents 这一类全局选项，来快速从一个项目入口出发，构建基于其上游依赖或下游依赖的 Project Graph（更详细地说，属于有向无环图） ，但确实用着比较麻烦（include / exclude，dependents / dependencies）。

实际上，这里的依赖关系，其最简本质也就是将一个有向无环图（DAG）进行拓扑排序的过程。对于部分 script，必须严格按照这一顺序来执行（如 start、build、test 等），否则会出现依赖引用的产物版本不一致等问题。

如果你只是需要任务调度能力，而不需要 lerna 的其他能力，可以使用一些轻量的替代方案，如 [ultra-runner](https://www.npmjs.com/package/ultra-runner) 等。

对于工程依赖，lerna 以及后面的 yarn workspace 都进行了更好的处理，如共同依赖地提升（hoist）等。当然，依赖提升等特性又带来了新问题，如幽灵依赖（Phantom Dependencies）使得包可以去访问未在 package.json 中声明的依赖（如依赖的依赖），以及 Doppelgangers 问题中被重复安装多次的依赖，但这些不是本文的重点，就不做展开讲解了。

除了 Lerna + Yarn Workspace 以外，新晋的包管理器 pnpm 也有着自己的 Monorepo 方案，同时由于其独特的依赖处理机制、强大的 filtering 语法（用 ... ^ 的简写形式代替了 dependents / dependencies，支持了基于 git 的 affected 检测等），许多知名开源项目都在从 lerna 迁移向 pnpm workspace，如 Vue、Vite 等等。我个人的项目也基本上全部完成了到 pnpm workspace 的迁移，如我的起手式项目 [starter-collections](https://github.com/LinbuduLab/starter-collections) ，为了获得更好的 pnpm workspace 下的开发体验，我还为它开发了个 VS Code 扩展 [pnpm-vscode-helper](https://github.com/LinbuduLab/pnpm-vscode-helper)。

在过去，Lerna + Yarn Workspace 的形式是社区中使用率最高的 Monorepo 方案（应该目前不需要加上之一）。在这一搭配中， lerna 负责任务调度，yarn workspace 负责依赖处理，看起来好像解决了所有问题，但程序员就是一个永远不会知足的团体，有一部分人开始思考，Monorepo 下的工作流是否还可以更丝滑？

#### 当下与未来：Monorepo Framework

要想获得 Monorepo 下更好的工作流体验，我们得先确认目前有哪些不好的部分？

* 任务调度虽然很棒，但实际上开发中可能下游的依赖项目改动频率比较低，虽然构建器可能有缓存功能（tsc 等），但是我更希望直接跳过未发生改动的项目，极致地压缩性能。
* 我需要在 package.json 中描述依赖关系，相当于我还是要有一点心智负担在，能否由 Task Runner 来自动分析依赖关系？
* 上面说了有些 script 是一定要按照拓扑排序来构建的，但有些也是不用的，比如 lint 这种，就可以无视依赖关系并行执行。这样我的任务执行是否能再快一些？
* 如果是大型的、多人甚至多团队协作的大型 Monorepo 项目，那各个团队的项目之间其实还是需要有一定的约束关系存在，比如我不希望别的团队在没经过允许的情况下去修改我的代码，也不希望未经过允许就直接使用我团队的基础库，然后出了问题又要找我修，我拒绝！
* 对于构建开销非常大的项目，如果多个团队成员使用的是同一个版本，那么其实只需要有一个人构建完毕，把产物上传到云端，其他人开始构建时直接下载此产物即可，这不比构建快多了？
* ......

而要想解决这些问题，只靠 Task Runner 可就不够了。我们需要的是一整套的、全生命周期覆盖的解决方案，我这里将其称为 Monorepo Framework（目前似乎没看到别人这么称呼，那我偷偷声明一下原创权利！），目前我认为可被称作 Framework 级别的解决方案主要有这么几个：

* Nx，我个人最推荐的一个解决方案。作者是前 Google 工程师，团队中也有许多专注于 Monorepo 的大厂成员。按照官方的叙述，Nx 吸收了许多 Google、Meta 内部 Monorepo 方案的优点。
* Turborepo，2021 年的新起之秀，原是个人项目，后被 Vercel 收购。我个人认为这是 Vercel 在前端工程中的进一步开疆拓土，现在你的框架、仓库管理、部署都可以只靠 Vercel 完成了。
* Rush，微软开源的 Monorepo 方案，我个人没有做过比较深入的了解，这里不做评论。
* 一些企业级的解决方案，如 Google 的 Bazel、Gradle 这种，这些和前端的关系不是太大，上手成本也较高，这里同样不做评论。

我个人比较想详细介绍一下 Nx 和 Turborepo，我们来一点点说一下上面解决的问题。

* 任务缓存。Nx 和 Turborepo 都支持了任务的缓存（Local Computation Caching），但实现方式略有差异。Nx 对缓存计算使用了类似 React （VDOM）那样的 diff 计算，性能方面要更好。
* 依赖关系分析。Nx 的依赖分析除了 package.json 以外，还会基于 AST 的分析来确定一个子项目的上下游依赖。而 Turborepo 则和此前的 workspace 方案类似，主要基于 package.json 中的 workspace: 协议。
* 可并行的任务调度。实际上这里的任务调度要比 lerna 这一类方案强大得多，如你可以更加自由的定义任务的执行方式、前置后置任务等等。而 Turborepo 比 Nx 强大的地方在于，通过 pipeline 配置的方式，支持了更好的任务链计算，如这张图片就能很好地说明其优势：

<figure><img src="https://picx.zhimg.com/80/v2-d664989763a450acccbfc8dc7cb983ba_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

实际上，Turborepo 的这一特色来自于微软的 Lage，一个介于 Task Runner 和 Framework 之间的方案。说它只是 Runner 吧，它又支持了云端产物缓存，说它是 Framework 吧，它又没有其他拿得出手的功能。

* 项目间引用约束。Nx 中支持为项目配置中添加 Tag，通过维护一份合法的 Tag 间引用关系以及 ESLint 插件配置，来实现项目之间的隔离，如 TeamA Tag 的项目不允许引用 TeamB Tag 项目。而 Turborepo 目前暂时不支持。
* 云端缓存。先说 Turborepo ，由于 Vercel 强大的 PaaS 能力，Turborepo 的构建缓存可以非常自然地融入其中，但缺点在于锁死了平台，对于部分大厂来说并不能接受将自己的应用代码存储在别的公司云端中。而 Nx 既提供了 Nx Cloud 这一方案，也提供了基于 Nx Cloud API 将云端缓存能力融入到内部 CI/CD 流水线的方案，河南拔智齿。如果这一方案能够在大厂中进行落地，那么其已经非常丝滑的构建体验又将再上一个等级。

除了这些能力以外，Monorepo Framework 其实还提供了许多贴心的功能。如 Nx 支持了 Angular 式的 Schematics 与 Builder，在 Nx 中这两个被称为 Generator 与 Executor。Generator 即快速生成项目起始模板或基础代码片段的能力，而 Executor 提供了项目的各个 script target 自定义执行的能力，如你可以基于内部的构建器提供 Executor。通常来说，Nx 中通过 Plugin 的形式来整合同一框架级别的 Plugin，如 @nrwl/react, @nrwl/nest 等等。这也是为何我更推崇 Nx 的原因，它允许你通过插件的形式非常自由地接入任何你想用但官方没有提供集成的框架，这些集成可以是社区新秀如 Vite、SvelteKit，可以是非 Web 项目的 Electron、Ionic 等，甚至可以是其他语言如 Go 。以上我们所讲述的就是目前的 Monorepo Framework 方案，不妨放飞你的想象力，想想未来的 Monorepo 方案又会如何演进？我个人大概有这么几点盼望：

* 和包管理器的深度集成，上面我们说到的这些方案和包管理器仍然是割裂的，比如要想使用 pnpm workspace 和 nx ，就需要启用 pnpm 的 shamefullyHoist，并且仍然可能会遇到问题。
* 和 Local Registry（如 Verdaccio）的深度集成，Monorepo 中开发项目时，如果就在 workspace 内部开发还好，而一旦引用项目在 workspace 外部，就又要回归到原始的 link 了。
* 更加开放的插件体系，Nx 做得挺好，但还不够好。
* 从 Polyrepo 到 Monorepo 更好、更稳定的迁移方案，比如给定几个仓库，能够自动地按照 package.json 进行依赖关系构建，自动引入对应框架的配置（甚至对于项目内定制配置，也可以用过类 nx run-commands 的方式进行迁移）。

很明显，Monorepo 方案还有很长的路可以走，也希望更多的团队与公司尝试拥抱 Monorepo方案。它不仅仅是一种项目管理方式，更是一种拥抱开放的态度。
