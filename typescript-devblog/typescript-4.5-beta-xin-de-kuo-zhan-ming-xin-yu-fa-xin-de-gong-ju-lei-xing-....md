---
description: '2021-10-03'
---

# TypeScript 4.5 beta：新的扩展名、新语法、新的工具类型...

TypeScript 4.5 已于 10.1 发布 beta 版本，本文将介绍部分其中值得关注的新特性与变更，如新增 `.mts` / `.cts` 扩展名、新的类型导入语法、新增内置工具类型等，你也可以阅读 [devblog](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5-beta/) 原文了解更多。

### Node ESM 支持 ECMAScript Module Support in Node.js

在 Node12 以后对 ESM 的支持逐渐平稳的今天，TS4.5也终于开始了对 Node 下的 ESM 相关支持，这应该是 4.5 版本最为核心的新特性了，由于原文篇幅过长，这里只简单介绍下相关细节：

* 新增了 `compilerOptions.module`：`node12` 与 `nodenext`
*   package.json `type`

    NodeJS中支持在 package.json 中设置 `type` 为 `module` 或 `commonjs` 来显式的指定 JavaScript 文件应该被如何解析。ESM 比之于 CJS，仍存在着一些显著的差异，如相对路径导入需要提供带扩展名的路径，即 `import "./foo.js"` 的形式。现在 TS4.5 对此也提供了相同的工作流，即 package.json 中的 `type` 字段现在也会被 TS 读取，来决定是否将其作为 ESM 解析。
*   新的文件扩展：`.mts` 与 `.cts`

    除了使用 `type` 字段来控制模块解析以外，你也可以显式的使用 TS4.5 新增的两个扩展名 `.mts` 与 `.cts` 来声明文件，就像 NodeJS 中一样，`.mjs`始终会被视作 ESM，而 `.cjs` 始终会被视作 `CJS`，而这两个新扩展名也会对应的编译到 `.d.mts` + `.mjs` 或 `.d.cts` + `.cjs` 的形式。
*   package.json中的 `exports` 与 `imports`：

    在简单的情况下，我们只要使用 `main` 字段来定义应用程序的入口即可，但如果想更精细的控制对用户暴露的文件，就需要使用 `exports` 与 `imports`了，我最早看见这种用法是在 [Astro](https://github.com/snowpackjs/astro/blob/main/packages/astro/package.json#L13) 中，它将 CLI 相关的代码移了出去，使得用户不能进行 Programmatic 接口进行相关定制（虽然我也不明白为什么要这么做，是因为还不稳定？）

    你可以在 [NodeJS文档](https://nodejs.org/api/packages.html#packages\_exports) 中找到更多对于这部分的相关说明，这里我们只做简要叙述。当你这么定义 `exports`：

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

    用户可以通过 `pkg` 引用 `pkg/main.mjs` 的内容，通过 `pkg/foo` 引用 `pkg/foo.js` 的内容，通过`pkg/dir/file.js` 引用 `pkg/dir` 下的 `file.js` , 而不能通过 `pkg/cli` 应用 `pkg/cli.js` 的内容，即使 `main.mjs` 中引用了它。

    > 另外，由于 [Self-referencing](https://nodejs.org/api/packages.html#packages\_self\_referencing\_a\_package\_using\_its\_name) 特性的存在，你也可以在这个包内部的文件中使用自己的包名来引用自身。
    >
    > 你可以在 [proposal-pkg-exports](https://github.com/jkrems/proposal-pkg-exports) 中查看这一提案的提出是为了解决哪些问题，以及更多相关信息。

    对于 `ESM` 和 `CJS` 的入口，你也可以通过这一方式来指定：

    ```json
    {
      "name": "pkg",
      "exports": {
        ".": {
          "import": "./esm/index.mjs",
          "require": "./cjs/index.cjs",
        }
      }
    }
    ```

    从而为 `import(pkg)` 和 `require(pkg)` 提供不同的入口。

    回到 TS 原本的逻辑，它会检查 `main`，以及其相关的类型文件（如 `./lib/main.js` 对应于 `./lib/main.d.ts`），或者通过 `types`获取声明文件地址（如果有的话）。类似的，现在如果你使用 `import`，它就会去 `import` 的地址寻找类型声明文件，反之则是 `require`，你仍然可以新增单独的 `types` 字段：

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
      // 兼容先前的版本
      "types": "./types/index.d.ts"
    }
    ```

### 支持从 node\_modules 加载 lib Supporting `lib` from `node_modules`

我们知道，tsconfig中 `compilerOptions.lib` 用于包含需要在编译时使用的语法或者 API，通常是DOM，ESNext ，WebWorker 这一类与语言以及环境有关的 API 声明，比如说，要使用 `Promise`，就需要 `ES2015`，要使用 `replaceAll`，就需要 `ESNext`。

这一种方式存在着一定的问题，难以进行细粒度的定制，比如我只需要 `DOM` 的一部分和 `ESNext` 的一部分。或者是在更新 TS 版本时其内置 `lib` 声明可能存在的 Breaking Change。你可能会想到，另一种管理全局类型的方式是 [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped)，即 `@types/node` 这一类 npm 包。因此 TS4.5 也支持了通过这一方式来显式的安装所需依赖，如 `@typescript/lib-dom` 就代表了原先的 `DOM`。

当你的 lib 中包含 DOM 时，TS会先在 `node_modules/@typescript/lib-dom` 这个位置查找是否有对应的包存在，而它在你的 `dependencies` 中声明实际上是这样的：

```json
{
 "dependencies": {
    "@typescript/lib-dom": "npm:@types/web"
  }
}
```

`npm:`这一协议实际上是 npm 提供的，类似的还有 `file:` 以及 `workspace:`(`yarn2 workspace`、`pnpm workspace`)，你所安装的实际上也是 [@types/web](https://www.npmjs.com/package/@types/web) 这个包。

对于解析逻辑发生的变更，详见 [compiler/program.ts](https://github.com/orta/TypeScript/blob/77c0e2e28d1ff69a718392add055b59c164d8cd3/src/compiler/program.ts#L2886)。

### 新的内置工具类型 Awaited The `Awaited` Type and `Promise` Improvements

TS 4.5 引入了新的工具类型 `Awaited`，表示一个 Promise 的 resolve 值类型，社区工具库早已存在类似功能的工具类型，如[type-fest](https://github.com/sindresorhus/type-fest) 中的 `PromiseValue`：

```typescript
export type PromiseValue<PromiseType> = PromiseType extends PromiseLike<infer Value> ? PromiseValue<Value> : PromiseType;
```

它的作用实际上即是递归的执行拆箱，提取Promise 值的类型。但不同于社区实现，官方的 `Awaited` 还被作为 `Promise.all` `Promise.race` 等相关方法的底层实现，如 TS4.5 以前的 `Promise.all` 方法，类型定义是这样的：

```typescript
interface PromiseConstructor {
		all<T>(values: readonly (T | PromiseLike<T>)[]): Promise<T[]>;
		//...省略无数重载
    all<T1, T2, T3, T4, T5, T6, T7, T8, T9, T10>(values: readonly [T1 | PromiseLike<T1>, T2 | PromiseLike<T2>, T3 | PromiseLike<T3>, T4 | PromiseLike<T4>, T5 | PromiseLike<T5>, T6 | PromiseLike<T6>, T7 | PromiseLike<T7>, T8 | PromiseLike<T8>, T9 | PromiseLike<T9>, T10 | PromiseLike<T10>]): Promise<[T1, T2, T3, T4, T5, T6, T7, T8, T9, T10]>;
}
```

而现在则是这样的：

```typescript
interface PromiseConstructor {
  all<T extends readonly unknown[] | []>(values: T): Promise<{ -readonly [P in keyof T]: Awaited<T[P]> }>;
}
```

### 基于模板字符串类型的类型收束 Template String Types as Discriminants

对于存在相同字段的接口，我们通常用类型守卫来显式的收束类型，如：

```typescript
export interface Success {
  type: string;
  body: string;
}

export interface Error {
  type: string;
  message: string;
}

export function isSuccess(r: Success | Error): r is Success {
  return "body" in r
}
```

使用 `is` 关键字能帮助编译器进一步的收束泛型到对应类型，可参考 [TypeScript的另一面：类型编程](https://juejin.cn/post/6885672896128090125) 或 [TypeScript的另一面：类型编程（2021重制版）](https://juejin.cn/post/7000360236372459527) 了解更多类型守卫、is关键字以及模板字符串类型相关。

现在对于模板字符串类型，TS 也提供了相关的支持：

```typescript
export interface Success {
  type: `${string}Success`;
  body: string;
}

export interface Error {
  type: `${string}Error`;
  message: string;
}

export function isSuccess(r: Success | Error): r is Success {
  return r.type === "HttpSuccess"
}
```

### 新的可用 module 配置`--module es2022`

TS 4.5 支持了新的 `compilerOptions.module` 配置：`es2022`，其主要特性就是 `top-level await`，这一特性其实在 `esnext` 中也可使用，但 `es2022` 属于第一个此提案被正式纳入的 ES 版本。另外，同属于本次新增的 `nodenext` 也包含了这一特性支持。

### 条件类型的尾递归省略 Tail-Recursion Elimination on Conditional Types

我们使用 TS 类型别名时，常常会遇到需要循环引用类型别名自身的情况，TS 编译器会检测到可能存在的无限嵌套情况并给出警告，如以下这种情况：

```typescript
type InfiniteBox<T> = { item: InfiniteBox<T> }

type Unpack<T> = T extends { item: infer U } ? Unpack<U> : T;

// 类型实例化过深，且可能无限。
// error: Type instantiation is excessively deep and possibly infinite.
type Test = Unpack<InfiniteBox<number>>
```

如果做过一些基于模板字符串类型的类型体操，你可能遇到过这种基于 `infer` 和 条件类型来提取字符串的情况：

```typescript
type TrimLeft<T extends string> =
    T extends ` ${infer Rest}` ? TrimLeft<Rest> : T;

// Test = "hello" | "world"
type Test = TrimLeft<"   hello" | " world">;
```

> 对于模板字符串，TS提供了 `Uppercase` `Lowercase` `Capitalize` 以及 `Uncapitalize` 这几种专用的工具类型

现在看起来是好的，但如果你在字符串的开头加入了大量的空格，可能就报错了，实际上这种对字符串做提取的操作是非常常见的，尤其是 URL 解析（参考 [farrow](https://github.com/farrow-js/farrow) 的核心特性）。

再回到 `TrimLeft` 本身的实现，你会发现它实际上属于尾递归的形式，即能够在每次递归的调用中立刻返回一个值，并且其返回值不会有额外的操作。这样一来，TS编译器就不需要去每次单独的创建中间变量，也就不再会触发警告了。

而反例则是去额外的使用了条件类型的判断结果，如

```typescript
type GetChars<S> =
    S extends `${infer Char}${infer Rest}` ? Char | GetChars<Rest> : never;
```

在这里，`Char` 与 `Rest` 被组合成了联合类型，这就将带来每一次的额外工作，因此仍然会在复杂度达到一定程度时被警告。

这也是 TS4.5 中引入的重要特性之一，如果条件类型的分支就只是简单的返回了另一个类型（自身，别的工具类型，泛型，infer提取值，等），那么 TS 就能减少许多不必要的中间工作，因此相比之前 “宽松” 多了。

* 递归的处理条件类型，由于是尾递归所以没问题
* 与循环引用自身不一样
* 检测到条件类型的分支仍然是条件类型时，智能组织

### 避免导入语句被省略 Disabling Import Elision

在 TypeScript 中，没有使用到的导入成员会被自动移除，如

```typescript
import { Foo, Bar } from "some-mod"

Foo()
```

其中的 `Bar` 将在编译时被移除，那如果存在部分情况 TS的内置检查策略不管用呢？比如使用了 `eval`：

```typescript
import { Foo, Bar } from "some-mod"

eval("console.log(Foo())")
```

这种时候我们就不希望导入成员被移除，TS4.5引入了新的编译时选项 `--preserveValueImports` ，来避免任何导入被移除。

这一特性还对 Vue、Svelte、Astro 这一类使用自定义文件（`.vue`/`.svelte`/`.astro`）的框架有着特殊的意义，通常其模板的编译是由自己处理的，而 script 部分的编译则由 TS 处理。这就使得模板部分对导入的使用无法被 TS 编译器感知到，需要额外的工作。

在先前的版本中 TS 还引入了 `--importsNotUsedAsValues` 选项来控制整条 import 语句的情况，其值包括：

* `remove`（默认），只有仅引入了类型的导入语句会被移除
* `preserve`，所有导入的值或类型没有被使用的导入语句都会被保留
* `error`，类似于 `preserve`，但是会在导入仅有类型时抛出错误

当 `--preserveValueImports` 和 `--isolatedModules` 一同使用时，必须确保类型导入是通过 type-only 的方式导入的，如 `import type` 与 `type` 修饰符。先来简单的介绍下 `--isolatedModules`：

如果你有一定的使用经验，一定知道当启用 `--isolatedModules` 时，每个文件都必须是模块（至少有个 `export {}`），这是因为对于 `ts-loader` `babel` `esbuild` 这一类的工具来说，它们通常是单个文件进行处理的（TypeScript的 `transpileModule` API 也是），不像 `tsc` 那样有预处理器收集源文件、生成编译上下文等的过程。

因此当启用 `--isolatedModules` 时存在着一定限制：

* 每个文件都必须是模块，即要么有 `import`，要么有 `export`
* 类型导入不能再导出（`re-export`），因为类型代码实际上在编译期会被移除掉，这样其他编译器感知不到这个问题，代码运行时就报错了。
* 对常量枚举（`const enums`）的导入、导出以及声明都是不被允许的，不同于普通枚举，常量枚举会在编译时直接被内联后抹除，即代码中使用 `SomeEnum.Foo` 的地方会被直接替换为枚举的值，这样单文件编译时除非常量枚举就定义在同一文件，否则根本无法获取其值。

### 新的类型导入语法 `type` Modifiers on Import Names

在 TS4.5 以前，我们可以这么来标识一条导入语句，其具名导入成员均为类型。

```typescript
import type { CompilerOptions } from "typescript"
import { transpileModule } from "typescript"
```

这么做其实不太美观，需要分成两个导入语句，如果强迫症犯了，你可能还要专门把文件的导入语句归类下，比如

```typescript
// 类型导入
import type { CompilerOptions } from "typescript"
import type { OpenDirOptions } from "fs-extra"
import type { Options } from "chalk"

// 实际导入
import { transpileModule } from "typescript";
import { opendirSync } from "fs-extra"
import chalk from "chalk"
```

但是现在，你可以直接给具名类型导入加上 `type` 修饰符，在一行导入里标识实际导入与类型导入。

```typescript
import { transpileModule, type CompilerOptions } from "typescript";
```

### 导入断言 Import Assertions

这一新特性来自于 提案 [proposal-import-assertions](https://github.com/tc39/proposal-import-assertions)，目前处于 Stage 3 阶段。其引入了新的语法 `import json from "./foo.json" assert { type: "json" };` 来显式的标识导入模块的类型。这一提案实际上大有可为，如配置 HTML 与 CSS Modules 实现 真·官方组件化，最初这一提案的目的是为了导入 JSON 文件，但现在它已经获得了独立提案：[proposal-json-modules](https://github.com/tc39/proposal-json-modules)。这一提案也可以用在：

* 重导入语句中，如`export { val } from './foo.js' assert { type: "javascript" };`
* 动态导入语句，如`await import("foo.json", { assert: { type: "json" } })`，在 TypeScript 4.5 中，专门新增了 `ImportCallOptions` 来作为动态导入第二个参数的类型定义。

另外，TC39提案必然会不断地融入TypeScript，成为新的特性，你可以阅读 [聊一聊进行中的TC39提案（stage1/2/3）](https://juejin.cn/post/6974330720994983950#heading-0) 这篇文章里一睹更多进行中的 TC39 提案。

### 更好的未解析类型提示 Better Editor Support for Unresolved Types

这一新特性主要是为未解析的类型声明新增 `/*unresolved*/` 的特性来提升使用体验：

<figure><img src="https://img.alicdn.com/imgextra/i3/O1CN01oodlyd1ktsqsM7yJp_!!6000000004742-2-tps-1202-248.png" alt=""><figcaption></figcaption></figure>

在这之前，未解析的类型声明只会被标记为`any`。

你可以在 [TypeScript 4.5 Iteration Plan](https://github.com/microsoft/TypeScript/issues/45418) 查看 4.5 版本的迭代计划，全文完，我们 TS4.6 见:-)
