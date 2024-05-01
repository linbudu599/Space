---
description: '2023-10-07'
---

# TypeScript 5.3 beta：Import Attributes 提案、Throw 表达式、类型收窄优化

TypeScript 已于 2023.10.03 发布 5.3 beta 版本，你可以在 [5.3 Iteration Plan](https://github.com/microsoft/TypeScript/issues/55486) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [JavaScript and TypeScript Nightly](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) ，并选择为项目使用 VS Code 的 TypeScript 版本，来更新内置的 TypeScript 支持：

<figure><img src="https://pic1.zhimg.com/80/v2-51b2192c31ee924c6bdcc87a92833d3a_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

本篇是笔者的第九篇 TypeScript 更新日志，上一篇是 「TypeScript 5.2 beta 发布：using 关键字、装饰器元数据、元组具名与匿名元素混用」，你可以在此账号的创作中找到（或在掘金/知乎搜索林不渡)，接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

总体来说，5.3 版本没有引入新的类型语法特性，主要的内容包括支持了两个 TC39 提案以及数个类型收窄相关的优化。

### Import Attributes 提案

此语法来自于 TC39 提案 [proposal-import-attributes](https://github.com/tc39/proposal-import-attributes)，在今年 3 月份的双月会议上进入到 Stage 3 阶段。这一提案的主要目的在于，在 import / export 语句中新增如下语法，用于为导入语句添加额外的描述：

```typescript
import json from "./foo.json" with { type: "json" };

import("foo.json", { with: { type: "json" } });

export { val } from './foo.js' with { type: "javascript" };
```

> 其中，json 类型的 Import Attributes 现在已经被拆分为一个独立的 Stage 3 提案，见 [proposal-json-modules](https://github.com/tc39/proposal-json-modules)。

这一提案的提出主要是为了解决导入文件和其 MIME 类型可能不一致的问题，如导入 JSON 时，MIME 类型意外返回了 text/javascript，那去执行 JSON 模块就会导致错误，因此我们需要一种独立于 MIME 之外，由 Client 指定导入文件类型而非 Server 的能力。

### resolution-mode 特性现已稳定

此前在 4.7 版本中，TypeScript 为三斜线指令支持了新的属性 resolution-mode 来控制 npm 包的解析方式：

```typescript
/// <reference types="pkg" resolution-mode="require" />
// or
/// <reference types="pkg" resolution-mode="import" />
```

本质上这修改了项目内统一的对文件的解析方式，比如你可以在项目内通过这种方式来指定使用 require 解析一个 ESM 的 npm 包。而上面的 Import Attributes 其实也是类似的能力，即它会“指导”当前运行时应该如何解析这个模块。因此，在 5.3 版本还支持了 Import Attributes 的 resolution-mode 配置：

```typescript
import type { TypeFromRequire } from "pkg" with {
    "resolution-mode": "require"
};

import type { TypeFromImport } from "pkg" with {
    "resolution-mode": "import"
};
```

实际上 4.7 版本当时就已支持这么做，但当时这个提案还叫 Import Assertion，用的语法还是 assert，当时的提案内容也不如现在完善，所以 TS 团队在当时仅仅将它添加到了 nightly 版本来进一步收集反馈。

### 类型收窄优化

#### switch(true)

5.3 版本优化了使用 switch(true) 时各个 case 分支的类型控制流分析，如以下的代码：

```typescript
function f(x: unknown) {
    switch (true) {
        case typeof x === "string":
            // 'x' is 'unknown' here.
            console.log(x.toUpperCase());
        case Array.isArray(x):
            // 'x' is 'unknown' here.
            console.log(x.length);
        default:
          // 'x' is 'unknown' here.
    }
}
```

此前这种写法内，各个 case 语句的 x 不会正常进行类型收窄，如 typeof x === "string" 成立时 x 应被收窄到 string 类型这样，5.3 版本已对此问题进行了修正。

#### 布尔值比较

此前版本中，如果将类型守卫的调用直接和布尔字面量值进行比较，类型守卫不会正常地进行类型收窄，如 isString(input) === true 这样的形式，5.3 版本对此问题进行了修正：

```typescript
function isString(x: any): x is string {
    return "toUpperCase" in x;
}

function someFn(x: unknown) {
    if (isString(x)) {
        console.log(x); // string
    }

    if (isString(x) === true) {
        console.log(x); // unknown before 5.3, string since 5.3
    }
}
```

说句题外话，TypeScript ESLint 中有条相关的规则是 [no-unnecessary-boolean-literal-compare](https://typescript-eslint.io/rules/no-unnecessary-boolean-literal-compare)，就用于检查将 boolean 类型的值和 true / false 字面量值进行比较。而我个人觉得，这就和类型断言是用 <> 还是 as 一样，只是个人风格喜好，而非绝对不推荐的代码范式。

### Super 访问检查

在 JavaScript 的 Class 中，你可以在子类中通过 super.parentMethod 方式来访问父类中的属性：

```typescript
class Base {
  someMethod() { }
}

class Derived extends Base {
  someMethod() { }
  
  someOtherMethod() {
    this.someMethod();
    super.someMethod();
  }
}
```

但有个问题是，这种方式只能够访问实例属性，即挂载在原型上的方法，对于静态方法是无法访问的，如：

```javascript
class Base {
  static someMethod = () => { }
}
```

这两种写法到 ES5 的编译产物如下：

```javascript
var Base = /** @class */ (function () {
    function Base() {
    }
    Base.prototype.someMethod = function () { };
    return Base;
}());

var Base = /** @class */ (function () {
    function Base() {
        this.someMethod = function () { };
    }
    return Base;
}());
```

此前，TypeScript 并不会在 super 访问中区别实例属性和静态属性：

<figure><img src="https://pic1.zhimg.com/80/v2-371fedd0cfc01fe605eb10515c7a4cc3_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

而现在它会了！

<figure><img src="https://picx.zhimg.com/80/v2-1991d77adf8637718bf920fb28591f9f_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

### Throw 表达式

此特性是对 TC39 提案 [proposal-throw-expression](https://github.com/tc39/proposal-throw-expressions) 的支持，其目前处于 stage 2 阶段。Throw Expression 提案允许你像使用表达式一样使用一个 throw 语句，包括在函数参数的默认值，函数返回值与三元表达式等：

```typescript
function save(filename = throw new TypeError("Argument required")) { }

lint(ast, {
  with: () => throw new Error("avoid using 'with' statements.")
  });

function getEncoder(encoding) {
  const encoder = encoding === "utf8" ? new UTF8Encoder()
    : encoding === "utf16le" ? new UTF16Encoder(false)
      : encoding === "utf16be" ? new UTF16Encoder(true)
        : throw new Error("Unsupported encoding");
}
```

这种方式进一步简化了抛出错误的逻辑，你可以直接在赋值候选的最后一处使用 Throw Expression，当前面的赋值候选全部失效访问到它时，就会抛出这里的错误。目前，Throw Expression 在 Babel 中被实现为一元表达式节点，即 `UnaryExpression`，类似于 `const visitor = !userLogin` 中的 `!userLogin` ，而 `!` 与 `throw` 则是 `UnaryExpression` 中的 `operator`。

### 其它优化

#### Inlay Hints 支持跳转至类型定义

Inlay Hints 作为 IDE 内语言服务的一部分，主要意义在于便捷地展示变量/参数/枚举等的当前类型，尤其适用于基于类型的控制流分析下类型的演化。以 TypeScript 为例，其在 VS Code 内内置的 Inlay Hints 相当强大：

<figure><img src="https://pic1.zhimg.com/80/v2-fb8177a212bda7748f81bbb422fa2bf8_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

而在 5.3 版本，Inlay Hints 现在支持了交互能力，你可以使用 Command + 点击的方式去访问 Hints 的参数/类型/变量等信息：

<figure><img src="https://picx.zhimg.com/80/v2-d05f86dc2350b924f22d63a0e0f9676b_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

#### JSDoc 解析策略

JSDoc 是 JavaScript 生态下的“从注释生成 API 文档”能力的实现，它提倡将代码功能的描述直接内联在代码中作为注释，并从符合规范的注释生成文档内容。对于 SDK 开发者，尤其是类似 Lodash 这种提供大量 utilities 函数的 SDK，使用这种代码与文档一体的方式能够相当有效地降低文档的维护成本——毕竟你也不想改完代码后还要在海量文档里找到对应方法复制粘贴吧？JSDoc 的大致语法是这样的：

```javascript
/**
 * 添加一本书。
 * @param {string} title - 书的标题。
 * @param {string} author - 书的作者。
 */
function addBook(title, author) { }
```

可以看到它包括了函数说明，参数类型与意义等信息。而在 TypeScript 中同样可以使用 JSDoc，如以下两种不同的标注类型的方式：

```javascript
let str: string;

/** @type {string} */
let str;
```

第二种写法其实通常会出现在 JS 代码中，通过配合 TSConfig 的 --checkJs 配置来实现对 JS 代码进行类型检查，这样一来就能实现一个看起来很完美的效果：既是纯纯的JS，又保留了类型检查。我们最常见的方式可能是在用 JS 定义各种配置文件时，由 JSDoc 提供类型信息：

<figure><img src="https://pic1.zhimg.com/80/v2-8af54e46ff547c0d1b6241ac711374eb_1440w.png?source=d16d100b" alt=""><figcaption></figcaption></figure>

同时，以社区项目为例，remark 团队维护的所有插件都使用的是 JSDoc，以 [remark-toc](https://github.com/remarkjs/remark-toc/blob/main/lib/index.js) 为例：

```javascript
/**
 * @typedef {import('mdast').Root} Root
 * @typedef {import('mdast-util-toc').Options} Options
 */

import {toc} from 'mdast-util-toc'

/**
 * Generate a table of contents (TOC).
 *
 * Looks for the first heading matching `options.heading` (case insensitive),
 * removes everything between it and an equal or higher next heading, and
 * replaces that with a list representing the rest of the document structure,
 * linking to all further headings.
 *
 * @param {Readonly<Options> | null | undefined} [options]
 *   Configuration (optional).
 * @returns
 *   Transform.
 */
export default function remarkToc(options) {
  /**
   * Transform.
   *
   * @param {Root} tree
   *   Tree.
   * @returns {undefined}
   *   Nothing.
   */
  return function (tree) { }
}
```

是的，在 JSDoc 里你仍然可以使用 TypeScript 的内置工具类型，还有使用 @typedef 定义全局类型，使用 import(lib).Type 引用三方类型等等。此前，在使用 tsc 编译 TS 代码时，它默认会对 JSDoc 进行解析，包括基于其进行的类型检查等等，来自 JSDoc 的类型信息都会被解析后保存在 AST 上。而在 TypeScript 5.3 版本，这一行为的默认表现被调整为不会再解析 JSDoc，从而在一定程度上降低编译耗时。除了默认行为的调整，TypeScript 将 JSDoc 相关的解析配置在 Compiler API 中也做了支持，以供各个工具按对应的需求进行调整：

```typescript
const host = ts.createCompilerHost(options);
host.jsDocParsingMode = ts.JSDocParsingMode.ParseForTypeInfo;
```

JSDocParsingMode 的各个枚举值介绍如下：

* JSDocParsingMode.ParseAll，解析所有文件的所有 JSDoc，并将信息呈现在 AST 中。
* JSDocParsingMode.ParseForTypeErrors，解析非 TS/TSX 文件中的所有 JSDoc，并会进行类型检查以获得类型的错误信息，这其实就是现在 tsc 的默认行为，参考 [tsc 的源码部分](https://github.com/microsoft/TypeScript/blob/release-5.3/src/executeCommandLine/executeCommandLine.ts#L794) 。
* JSDocParsingMode.ParseForTypeInfo，类似于上一个，但仅会提取类型信息，不会进行类型检查。
* JSDocParsingMode.ParseNone，跳过所有 JSDoc 解析。

全文完，我们 TS 5.4 见 :)
