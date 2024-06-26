---
description: '2022-04-20'
---

# TypeScript 4.7 beta：NodeJs 的 ES Module 支持、新的类型编程语法、类型控制流分析增强等

TypeScript 已于 2022.06.21 发布 4.8 beta 版本，你可以在 [4.8 Iteration Plan](https://github.com/microsoft/TypeScript/issues/49074) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [JavaScript and TypeScript Nightly](http://link.zhihu.com/?target=https%3A//marketplace.visualstudio.com/items%3FitemName%3Dms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第三篇 TypeScript 更新日志，上一篇是 「TypeScript 4.6 beta 发布：递归类型检查增强、参数的控制流分析支持、索引访问的类型推导」，你可以在此账号的创作中找到。在前一篇的经验上，笔者将进一步的完善文章的描写风格，包括部分 feature 的历史背景、实际应用以及适当扩展。接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

### 上版本回顾

TypeScript 4.6 版本的工作重心再次回到了类型能力这一部分，包括增强了启发式地递归类型检查、支持了索引访问类型地类型推导、参数类型地控制流分析支持等，我们来简单地回顾一下。如果想要详细地了解上一版本的变更，请前往笔者的专栏阅读。

#### 启发式的递归类型检查

假设我们有两个深层嵌套的工具类型：

```typescript
interface Foo<T> {
  prop: T;
}

declare let x: Foo<Foo<Foo<Foo<Foo<Foo<string>>>>>>;
declare let y: Foo<Foo<Foo<Foo<Foo<string>>>>>;

x = y;
```

我们能明显确定这里的两个类型并不兼容，但这里实际上并不会报错。这是因为对于这一类深度嵌套的情况，TypeScript 会使用启发式的递归检查，即，执行一定深度的展开检查，如果还没完事就判定这是一个无限循环，则认为两个类型是兼容的，此策略称为**启发式的递归类型检查**。这一策略能够一定程度下提升性能，但由于其关注的是嵌套展开的情况，而非实际声明的情况，就会导致上面这种进行一定深度检查后错误地认为两个类型兼容的情况。

4.6 版本中增强了这一策略，不再关注 **结构的泛型参数中引用了结构进行套娃** 这种来自于明确指定的特殊情况，即，关注点现在变成了嵌套层级。明显体现了其效果的则是一些 DefinitelyTyped Package 的类型检查工作减少了 50% 以上的成本，如 @types/yup 以及 @types/redux-immutable 等

#### 索引访问的类型推导

对于索引类型、索引访问类型、索引签名类型，请参阅专栏中 4.6 版本更新日志的详细介绍。

考虑以下示例：

```typescript
type UnionRecord =
  | { kind: 'n'; v: number; f: (v: number) => void }
  | { kind: 's'; v: string; f: (v: string) => void }
  | { kind: 'b'; v: boolean; f: (v: boolean) => void };

type VTypes = UnionRecord['v'];
```

这里 `VTypes` 能够被正确的推导为 `string | number | boolean`，但这一推导结果在以下就将导致一个错误：

```typescript
function processRecord(rec: UnionRecord) {
  rec.f(rec.v); // Error, 'string | number | boolean' not assignable to 'never'
}
```

我们知道，对于可辨识联合类型的各个类型分支，其每一分支的属性类型之间应当是独立的，而同一分支内部的类型又应该关联。而在这里，很明显三个分支的类型并没有独立起来，否则每一分支的 f 入参类型应当对应于此分支的 v 类型。

在 4.6 版本前，你可以通过泛型或额外类型守卫的方式来显式的纠正类型地控制流分析，而在 4.6 版本中，对于可辨识联合类型的分析得到了优化，上面的代码类型现在能够被正确地推导。

#### 参数的类型控制流分析

> 关于 TypeScript 的类型控制流分析，同样参考笔者知乎/掘金专栏中的文章：「TypeScript 中的控制流分析演进」。

这一能力支持了在函数中，对参数类型的控制流分析：

```typescript
type Args = ['a', number] | ['b', string];

const f1: Func = (kind, payload) => {
  if (kind === 'a') {
    payload.toFixed(); // 'payload' narrowed to 'number'
  }
  if (kind === 'b') {
    payload.toUpperCase(); // 'payload' narrowed to 'string'
  }
};
```

控制流分析实际上是非常常见又非常容易被忽略的点，如果你有兴趣，不妨阅读上面提到的文章来稍微深入了解下。

除以上三个类型能力增强以外，4.6 版本还支持 Class 构造函数 中在 super() 之前去执行代码（当然，不能访问 this）、新的性能分析工具 **TypeScript Trace Analyzer** 以及 JavaScript 文件的语法检查等。

### TypeScript 4.7 beta 综述

4.7 beta 版本是我目前印象中比较“庞大”的一个版本，其包含了部分来自于之前版本的未尽事业、新的类型编程语法、新的关键字、新的 Compiler Options、类型推导能力增强等等。也因此，在 4.7 beta 与 4.7 正式版本之间可能会存在一定差异，另外正式版本的发布大概率也会需要更长的时间。

4.7 beta 主要包含以下部分的更新：

* NodeJS 中的 ES Module 支持
* 模块检查控制
* 计算属性的类型控制流分析支持
* 对象内函数类型推导增强
* 泛型实例化表达式
* infer 关键字的 extends 约束
* 类型参数变化标记
* 对 # 声明私有字段的 typeof 支持
* 自定义模块解析策略
* 模块解析策略
* 导入语句的组织优化
* 对象方法的补全支持
* 破坏性变更

### NodeJs 中的 ES Module 支持 ECMAScript Module Support in Node.js

这一特性实际上在 4.5 版本就已经出现在 DevBlog 中，但由于其影响面较广，在当时只是被作为预览版本用于收集反馈和进行调整，推迟到了现在。最终 4.5 版本只引入了从属于此特性的一小部分（`--module es2022` 配置）。我在此前的文章中已经介绍过这一特性的大部分内容（参见 [TypeScript 4.5 发布：新的扩展名、新语法、新的工具类型...](https://juejin.cn/post/7014770180421058590#heading-0)）。

这一特性主要是为了支持 NodeJs 下 ES Module 的 TypeScript 开发能力，包括新增了两个新的 Compiler Options 的 module 配置：`node12` 与 `nodenext`（node12 是 ESM 开始在 NodeJs 中完整实现的版本）。

NodeJs 支持在 package.json 中设置 `type` 为 `module` 或 `commonjs` 来显式的指定文件应该被如何解析，而 ESM 比之于 CJS，在使用方面存在着一些显著的差异，如：

* 相对路径导入需要提供带扩展名的路径，即 `import "./foo.js"` 的形式。
* 无法使用 `__dirname`， `__filename`，`require` 这些全局的变量或方法

因此在 4.7 版本，TypeScript 也将会读取这一配置字段来决定是否将文件作为 ESM 解析，以及如何查找这一文件导入的模块、构建产物是否使用 ESM 等。

同时，对于路径需要携带扩展名这一点，现在对于使用 ESM 的 TypeScript 文件同样需要显式的注明：

```typescript
// ./bar.ts
import { helper } from './foo.js'; // works in ESM & CJS

helper();
```

除了使用 `type` 字段来控制模块解析以外，你也可以使用本次新增的两个文件扩展名 `.mts` 与 `.cts` 来声明文件，就像 NodeJS 中一样，`.mjs`始终会被视作 ESM，而 `.cjs` 始终会被视作 `CJS`，而这两个新扩展名也会对应的编译到 `.d.mts` + `.mjs` 或 `.d.cts` + `.cjs` 的形式。

在简单的情况下，我们只需要使用 `main` 字段来定义应用程序的入口即可，但如果想更精细的控制对用户暴露的文件，就需要使用 `exports` 与 `imports`了，我最早看见这种用法是在 [astro](https://github.com/withastro/astro/blob/main/packages/astro/package.json#L26) 中，它没有将 CLI 相关的代码如 dev、serve 等命令的实际执行方法导出，使得用户不能使用 Programmatic API 进行相关定制。

你可以在 [NodeJS 文档](https://nodejs.org/api/packages.html#packages\_exports) 中找到更多对于这部分的相关说明，这里我们只做简要叙述。当你这么定义 `exports`：

```json
{
  "name": "pkg",
  "exports": {
    ".": "./main.mjs",
    "./foo": "./foo.js",
    "./dir": "./some-dir"
  }
}
```

用户可以通过 `pkg` 引用 `pkg/main.mjs` 的内容，通过 `pkg/foo` 引用 `pkg/foo.js` 的内容，通过`pkg/dir/file.js` 引用 `pkg/dir` 下的 `file.js` , 而不能通过 `pkg/cli` 应用 `pkg/cli.js` 的内容，即使 `main.mjs` 中引用了它，这样就实现了精确的导入控制。

> 另外，通过 [Self-referencing](https://nodejs.org/api/packages.html#packages\_self\_referencing\_a\_package\_using\_its\_name) 特性，你也可以在这个包内部的文件中使用自己的包名来引用自身。
>
> 你可以在 [proposal-pkg-exports](https://github.com/jkrems/proposal-pkg-exports) 这个仓库中，查看这一提案的提出是为了解决哪些问题，以及更多相关信息。

对于 `ESM` 和 `CJS` 的入口，你也可以通过这一方式来指定：

```json
{
  "name": "pkg",
  "exports": {
    ".": {
      "import": "./esm/index.mjs",
      "require": "./cjs/index.cjs"
    }
  }
}
```

从而为不同的调用方式： `import(pkg)` 和 `require(pkg)` 提供不同的入口。

回到 TS 原本的逻辑，它会检查 `main`，以及其相关的类型文件（如 `./lib/main.js` 对应于 `./lib/main.d.ts`），或者通过 `types`获取声明文件地址（如果有的话，并且如果声明了此属性，就不会再有前面的查找逻辑）。

类似的，现在如果你使用 `import`，它就会去 `import` 的地址寻找类型声明文件，反之则是 `require`，你仍然可以新增单独的 `types` 字段：

```json
{
  "name": "pkg",
  "type": "module",
  "exports": {
    ".": {
      "import": {
        "types": "./types/esm/index.d.ts",
        "default": "./esm/index.js"
      },
      "require": {
        "types": "./types/commonjs/index.d.cts",
        "default": "./commonjs/index.cjs"
      }
    }
  },
  "types": "./types/index.d.ts",
  "main": "./commonjs/index.cjs"
}
```

* TypeScript 会在使用 ESM 导入时去 `import.types`指定的位置查找类型文件,而在 CJS 导入下去 `require.types` 查找类型文件。而 `default` 字段则是 NodeJs 消费的。
* 独立的 `types` 字段用于兼容先前版本的 TypeScript。
* 独立的 `main` 字段用于兼容先前版本的 NodeJs（注意区分 `main` 与`module`）

当仅有一份类型声明时，你也可以进行简化：

```json
{
  "name": "pkg",
  "exports": {
    ".": {
      "import": "./esm/index.mjs",
      "require": "./cjs/index.cjs",
      "types": "./types/index.d.ts"
    }
  },
  "types": "./types/index.d.ts"
}
```

### 模块检查控制 Control over Module Detection

默认情况下，TypeScript 会在检测到文件中存在着 Import/Export 语句时将此文件视为一个模块，否则将其视为一个应用于全局的文件。这一行为看起来似乎没什么问题，但考虑到 NodeJs 中对模块的定义是入口文件使用 `.mjs`，包的 package.json 中声明了 `"type": "module"`，以及在 React 项目中如果配置了 `--jsx react-jsx`，那么实际上所有的 `.jsx/.tsx` 文件中都隐式地包含了一行 React 的导入，这两种情况都意味着 TypeScript 的模块检查策略需要进一步地增强。

因此，4.7 版本中引入了新的配置 `moduleDetection.moduleDetection` （非笔误）来控制模块的检查策略，其配置值包括：

* `"auto"`，默认值，此时 TypeScript 在检查模块时除了检查 import 与 export 语句以外，还会在 `--module nodenext` 或 `--module node12` 时检查 package.json 中的 `type` 是否被设置为 `"module"`，以及在 `--jsx react-jsx` 下检查当前文件是否是 JSX 文件。
* `"force"`，此选项会强制将所有的文件视为模块，而不再关心 `module`、`jsx`、`moduleResolution` 这些配置。
* `"legacy"`，此选项即是 4.7 版本以前的默认解析行为，即仅检查 import / export 语句来确定文件是否是一个模块。

### 计算属性的类型控制流分析 Control-Flow Analysis for Computed Properties

继 4.6 版本以后，4.7 版本在类型控制流分析上再次迈出了一步。本次支持的是计算属性（即 `obj['key']` 这样的属性访问方式）的类型控制流分析。考虑以下代码：

```typescript
const key = Symbol();

const numberOrString = Math.random() < 0.5 ? 42 : 'hello';

let obj = {
  [key]: numberOrString,
};

if (typeof obj[key] === 'string') {
  let str = obj[key].toUpperCase();
}
```

在 4.7 版本以前， `typeof obj[key] === "string"` 成立后的语句块中，`obj[key]` 的类型并不会被收窄到 `string`。而在 4.7 版本引入了对计算属性的类型控制流分析支持后，这段代码现在可以正常地工作了。

同时，结合 2.7 版本引入的 [Strict Property Initialization, --strictPropertyInitialization](https://www.typescriptlang.org/tsconfig#strictPropertyInitialization) 配置，现在 Class 中计算属性也可以享受到赋值检查，如：

```typescript
const key = Symbol();

class C {
  [key]: string;

  constructor(str: string) {
    // 并没有进行赋值
  }

  screamString() {
    // 4.7 版本以前并不会报错
    return this[key].toUpperCase();
  }
}
```

### 对象中的函数类型推导增强 Improved Function Inference in Objects and Methods

4.7 版本还增强了定义在对象内部函数的类型推导能力，直接说有点绕，看一个例子：

```typescript
declare function f<T>(arg: {
  produce: (n: string) => T;
  consume: (x: T) => void;
}): void;

// Works
f({
  produce: () => 'hello',
  consume: (x) => x.toLowerCase(),
});

// Works
f({
  produce: (n: string) => n,
  consume: (x) => x.toLowerCase(),
});
```

这两个调用都是正常的，TypeScript 能够从 produce 函数的返回值推导出泛型参数 T 的类型，并应用到 consume 函数的入参类型中。而以下几个例子就不行了：

```typescript
f({
  produce: (n) => n,
  consume: (x) => x.toLowerCase(),
});

f({
  produce: function () {
    return 'hello';
  },
  consume: (x) => x.toLowerCase(),
});

f({
  produce() {
    return 'hello';
  },
  consume: (x) => x.toLowerCase(),
});
```

在第一处，produce 的入参类型并没有成功地传递给返回值类型。而在第二、第三个，produce 函数的返回值类型没有从其内部推导得到，仍然是默认的 unknown 类型。

在 4.7 版本，这种情况下的函数类型推导现在可以正确地从入参类型、内部逻辑（return 语句）等进行类型地推导。

### 泛型实例化表达式 Instantiation Expressions

毫不夸张的说，泛型的实例化表达式是本次更新我最期待的功能之一，它支持了对泛型的预填充而无需实际调用。举个栗子，假设我们要创建一个键类型为 string，键值类型为 Error 的 Map，通常会这么做：

```typescript
const errorMap: Map<string, Error> = new Map();
```

或者将这个 Map 类型抽离为一个类型别名：

```typescript
type ErrorMapType = Map<string, Error>;
```

两种做法都是在定义时的类型参数填充，且变量的类型是在实际调用时才确认的。而使用泛型实例化表达式，我们可以做到无需调用的情况下预先填充类型参数：

```typescript
// 注意，这里不是类型别名
const ErrorMap = Map<string, Error>;

const errorMap = new ErrorMap();
```

很明显，实例化表达式提供了比类型别名更自然的复用能力，我们是实例化已经填充完毕类型参数的 ErrorMap，而不是实例化一个普通的 Map 再把它的类型注释为 ErrorMap 类型，也不是通过继承于 Map 的派生类，如：

```typescript
class ErrorMap extends Map<string, Error> {}
```

一个更常见的场景是对接受泛型的函数按场景进行对应的实例化，如：

```typescript
function asFEEngineer<T>(value: T) {
  return { value };
}
```

这个函数只能确定是一个前端工程师，而不能确定其具体的方向如移动端，架构，NodeJs 等等，有了实例化表达式，我们可以通过预填充泛型参数的方式来实现不同场景的对应实例：

```typescript
const asMobile = asFEEngineer<"mobile">;
const asNodeJs = asFEEngineer<"nodejs">;
const asInfra = asFEEngineer<"infra">;
```

每一个函数除了泛型参数已固定以外，和原本的函数完全一致：

```typescript
const mobileFEEngineer = asMobile('mobile');
```

另外，由于实例化表达式的本质仍然是表达式，它也支持被作为 typeof 的输入，如：

```typescript
type StringBoxMaker = typeof asFEEngineer<"mobile">;  // (value: "mobile") => { value: "mobile" }
type ErrorMapConstructor = typeof Map<string, Error>;  // new () => Map<string, Error>
```

你可以阅读 [#47607](https://github.com/microsoft/TypeScript/pull/47607) 来了解更多细节。

### infer 的 extends 约束支持 `extends` Constraints on `infer` Type Variables

在 TypeScript 的类型编程中，条件类型是最重要的基础概念之一，我们可以使用它来判断类型的兼容性、收窄或映射一组联合类型、配合 infer 提取类型片段（如，数组的元素类型，函数的参数类型，模板字符串类型的某一部分）等。其中，结合 infer 地使用也相当广泛，比如我们可以提取数组/元组的首个字符串类型成员：

```typescript
type FirstString<T> = T extends [infer S, ...unknown[]]
  ? S extends string
    ? S
    : never
  : never;

// string
type A = FirstString<[string, number, number]>;

// "hello"
type B = FirstString<['hello', number, number]>;

// "hello" | "world"
type C = FirstString<['hello' | 'world', boolean]>;

// never
type D = FirstString<[boolean, number, number]>;
```

这个工具类型比较简单，首先使用 infer 匹配第一个元素类型，如果此类型是 string 则返回它，否则返回一个 never 。

如果你还没有习惯 TypeScript 的类型编程模式，你可能会想到这里是否还能更简单一些，比如在 infer 提取时就声明一个约束（类似于泛型约束那样），确保只会在这个位置的类型满足条件时才返回此类型？

4.7 版本支持了 infer 关键字的 extends 约束能力，这一能力能够大大简化许多现存工具类型/类型体操实现的条件语句判断，如上面的例子可以简化为：

```typescript
type FirstString<T> =
    T extends [infer S extends string, ...unknown[]]
        ? S
        : never;
```

当占位变量 S 匹配到一个类型时，它会确保条件语句在此类型符合约束时才满足（即走左侧的逻辑）。

如果你有兴趣，不妨翻阅 type-fest、ts-tool-belt 这些工具类型库，或 type-challenges 的题目解析，来看看哪些工具类型的实现可以使用此方式来进行优化。

### 类型参数的变化（协变、逆变）标记 Optional Variance Annotations for Type Parameters

> 这一部分的阅读可能需要你对 TypeScript 中的协变与逆变有一定了解，贴心的笔者在过往的专栏里已经发表过相关文章：「知其然，知其所以然：TypeScript 中的协变与逆变」，这一部分的讲解也部分来自于此文章。

考虑以下代码：

```typescript
interface Animal {
  animalStuff: any;
}

interface Dog extends Animal {
  dogStuff: any;
}

// ...

type Getter<T> = () => T;

type Setter<T> = (value: T) => void;
```

* `Getter<Animal>` 与 `Getter<Dog>` 之间的类型兼容性是如何的？
* `Setter<Animal>` 与 `Setter<Dog>` 之间的类型兼容性是如何的？

如果 `Getter<Dog> ≼ Getter<Animal>` 成立（`A ≼ B` 表示 A 是 B 的子类型），由于函数的返回值遵循协变（_**covariance**_），我们知道只需要 `Dog ≼ Animal` 成立即可，而这很明显是成立的。

如果 `Setter<Dog> ≼ Setter<Animal>` 成立，函数的参数类型遵循逆变（_**contravariance**_），需要 `Animal ≼ Dog` 成立，而这很明显是不行的。

在过去，我们只能通过已经确定的固定规律来判断协变与逆变分别在哪种情境下发生（参数逆变，返回值协变，部分内置方法双变（_**Bivariant**_），接口内部使用 `property` 方式定义的函数执行严格的协变与逆变检查，blabla...），4.7 版本则引入了新的关键字 `in` 与 `out` ，来标识此处的类型参数遵循协变或者逆变：

```typescript
type Getter<out T> = () => T;
type Setter<in T> = (value: T) => void;

type Provider<out T> = () => T;
type Consumer<in T> = (x: T) => void;
type Mapper<in T, out U> = (x: T) => U;
type Processor<in out T> = (x: T) => T;
```

`out` 标记这个类型参数使用协变检查，而 `in` 则标记其为逆变检查。这两个关键字的命名也来自于它们的实际场景，即在函数参数类型（输入）使用协变，返回值类型（输出）使用逆变。

你也可以同时使用这两个关键字来标记一个类型参数为不变（_**invariant**_），在这种情况下泛型参数之间必须是同一个类型（或者在结构化类型系统下能够被认为是同一个类型）：

```typescript
interface State<in out T> {
    get: () => T;
    set: (value: T) => void;
}
```

此时，`State<Dog>` 和 `State<Animal>` 之间就不存在可比较性。

既然这一关键字引入了新的约束支持，在约束不满足时的报错信息也是需要的：

```typescript
interface State<out T> {
    //          ~~~~~
    // error!
    // Type 'State<sub-T>' is not assignable to type 'State<super-T>' as implied by variance annotation.
    //   Types of property 'set' are incompatible.
    //     Type '(value: sub-T) => void' is not assignable to type '(value: super-T) => void'.
    //       Types of parameters 'value' and 'value' are incompatible.
    //         Type 'super-T' is not assignable to type 'sub-T'.
    get: () => T;
    set: (value: T) => void;
}
```

这一能力的引用除了上述这些能够让框架/工具库的作者进一步增强类型约束以外，还能显著提升 TypeScript 编译器的精确程度与类型检查的速度，你可以参考 [#48240](https://github.com/microsoft/TypeScript/pull/48240) 了解更多信息。

### 对#声明私有字段的 typeof 支持 `typeof` on `#private` Fields

在 TypeScript 中支持通过 `private` 关键字与 `#` 语法来标识类的成员为私有的，二者表现基本一致：

```typescript
class Example {
  #esPrivateProp = 'hello';
  private tsPrivateProp = 'hello';
}
```

需要注意的是，对于 `#` 语法，整个属性的名字包含 `#` ，即完整的 Identifier 应该是：`#esPrivateProp`。

在 TypeScript 4.7 以前，你无法对使用 `#` 声明的私有成员使用 typeof 操作符：

```typescript
class Example {
  #esPrivateProp = 'hello';
  private tsPrivateProp = 'hello'

  constructor() {
    const p: typeof this.#esPrivateProp = 'world';
    const p1: typeof this.tsPrivateProp = 'world';
  }
}
```

这里的 p 变量类型声明会报错：`Identifier expected`。原因是在 TypeScript 的 AST 中，`#` 属性使用 `PrivateIdentifier`，而非正常的 `Identifier`。两种声明的 AST 结构如下：

```bash
# #声明
PropertyDeclaration
|- PrivateIdentifier
|- StringLiteral

# private 声明
PropertyDeclaration
|- PrivateKeyword
|- Identifier
|- StringLiteral
```

而在 `typeof this.#esPrivateProp` 这一语句的 AST 结构中， PrivateIdentifier 不会被识别为合法的 Identifier：

```bash
TypeQuery
|- QualifiedName
|--- Identifier >>> this
|--- Identifier >>> 这里应当是 "#esPrivateProp"，但实际为 ""
```

在 4.7 版本中对 `PrivateIdentifier` 的识别做了支持，即 `typeof this.#esPrivateProp` 这一类语句现在可以正确运行了。

### 自定义模块解析策略 Resolution Customization with `moduleSuffixes`

此特性新增了 `moduleSuffixes` 这一 Compiler Options 来自定义模块的解析策略，如以下的配置：

```json
{
  "compilerOptions": {
    "moduleSuffixes": [".ios", ".native", ""]
  }
}
```

在使用以下导入时：

```typescript
import * as foo from './foo';
```

TypeScript Compiler 会优先查找 `foo.ios.ts`，`foo.native.ts`，最后才是 `foo.ts`。

配置中的 `""` 一项用于将无额外后缀的模块名（即 `foo.ts`）也纳入解析范围，同时它也是未显式配置时的默认值。

对于 React Native 项目，可以通过这一配置来为每一个平台对应的代码使用独立的配置文件以及 `moduleSuffixes` 配置。对于 Angular 项目，则可以通过这一配置来确保项目文件按照功能做了精确地命名，如：`validation.pipe.ts` 与 `user.service.ts` 等。

### 模块解析模式 resolution-mode

在 ES Module 的 Module Resolution 下，import 语句的解析会由实际使用的语法来决定，而现在我们可以更进一步，在 ES Module 中去使用来自于 CommonJS 导入的类型定义。

你可以通过 `import type` 语句中的 `import assertion` 语法来指定类型的解析模式：

```typescript
import type { TypeFromRequire } from "pkg" assert {
    "resolution-mode": "require"
};

import type { TypeFromImport } from "pkg" assert {
    "resolution-mode": "import"
};

export interface MergedType extends TypeFromRequire, TypeFromImport {}
```

在这段代码中，`TypeFromRequire` 与 `TypeFromImport` 会分别根据模块为 CommonJS 与 ES Module 提供的的类型导入入口来解析。

> 另外，Import Assertion 并不是一个全新的语法，它实际上是一个已经进入 Stage 3 的 TC39 提案，见 [proposal-import-assertions](https://github.com/tc39/proposal-import-assertions)。

或者，你也可以使用 `import()` 语法（不同于 Dynamic Import）：

```typescript
export type TypeFromRequire =
    import("pkg", { assert: { "resolution-mode": "require" } }).TypeFromRequire;

export type TypeFromImport =
    import("pkg", { assert: { "resolution-mode": "import" } }).TypeFromImport;

export interface MergedType extends TypeFromRequire, TypeFromImport {}
```

对于如框架内置类型定义（如 NextJs、Vite）的场景，resolution mode 也支持了通过三斜线指令（Triple-slash reference directives）来定义：

```typescript
/// <reference types="pkg" resolution-mode="require" />

/// <reference types="pkg" resolution-mode="import" />
```

### 导入语句的组织优化 Groups-Aware Organize Imports

TypeScript 会自动在编译产物中的导入语句进行组织，但这一组织形式太过简单，如按照 Module Specifier （即要导入模块的标识）简单进行排序，这一排序过程往往不会自动地识别注释语句，如以下的代码：

```typescript
// local code
import * as bbb from './bbb';
import * as ccc from './ccc';
import * as aaa from './aaa';

// built-ins
import * as path from 'path';
import * as child_process from 'child_process';
import * as fs from 'fs';

// some code...
```

在编译产物中的导入语句组织会是这样的形式：

```js
// local code
import * as child_process from 'child_process';
import * as fs from 'fs';
// built-ins
import * as path from 'path';
import * as aaa from './aaa';
import * as bbb from './bbb';
import * as ccc from './ccc';
```

可以看到编译产物的导入语句分组并没有遵循我们已经标记好的注释分组，因此在 4.7 版本中这也得到了优化，改善后的编译产物会是这样的：

```js
// local code
import * as aaa from './aaa';
import * as bbb from './bbb';
import * as ccc from './ccc';

// built-ins
import * as child_process from 'child_process';
import * as fs from 'fs';
import * as path from 'path';
```

### 对象方法的补全支持 Object Method Snippet Completions

对于使用对象字面量声明的方法，TypeScript 现在支持提供 snippet（代码片段）来一次性补全整个方法签名，示例：

<figure><img src="https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2022/04/object-method-completions-4-7.gif" alt=""><figcaption></figcaption></figure>

### 破坏性变更

#### 只读元组

在 TypeScript 中，通常我们认为元组是定长的数组，在这种情况下其 length 属性是固定的。但其实还存在着特殊的情况，如元组中的部分元素是可选的，或直接是一个开放式的元组，如：

```typescript
type OptionalElementTuple = [number, string?];

type OpenEndTuple = [number, ...string[]];
```

在这种情况下，其长度不再固定。

但是，一旦这个元组被标记为 readonly，那么其长度就应当也被标记为 readonly，等同于其 length 属性被标记为 readonly，而在 4.7 版本以前并没有此限制：

```typescript
declare const x: readonly [number?];
x.length = 0; // 正常
declare const y: readonly [number, ...number[]];
y.length = 0; // 正常
```

因此，在 4.7 版本中对这一问题进行了改进，现在只读元组的 length 属性也将是 readonly 的。

#### 其他

* 内置的类型定义（`lib.d.ts`）变更，devblog 中并没有给出具体的更新内容。
* 类型参数的兼容性，现在在启用 `strictNullChecks` 的情况下，无默认值的泛型参数不能分配给类型 `{}`。

你可以在 [TypeScript 4.7 Iteration Plan](https://github.com/microsoft/TypeScript/issues/48027) 查看 4.7 版本的迭代计划，预计在 5.6 发布 RC 版本，在 5.24 发布正式版本。

全文完，我们 TS 4.8 见。 :-)
