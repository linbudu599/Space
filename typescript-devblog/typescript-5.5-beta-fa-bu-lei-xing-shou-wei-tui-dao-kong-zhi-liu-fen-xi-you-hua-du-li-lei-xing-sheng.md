# TypeScript 5.5 beta 发布：类型守卫推导、控制流分析优化、独立类型声明等

TypeScript 已于 2024.4.25 发布 5.5 beta 版本，你可以在 [5.5 Iteration Plan](https://github.com/microsoft/TypeScript/issues/57475) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) ，并选择为项目使用 VS Code 的 TypeScript 版本(cmd + shift + p， 输入 select typescript version)，来更新内置的 TypeScript 支持。

本篇是笔者的第十一篇 TypeScript 更新日志，上一篇是 「TypeScript 5.4 beta: NoInfer 类型、闭包类型分析优化、条件类型判断优化等」，你可以在此账号的创作中找到（或在掘金/知乎/Twitter搜索**林不渡**)，接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

### 隐式类型守卫 Inferred Type Predicates

现在，TypeScript 会在符合条件时为你的函数返回值类型添加隐式的类型守卫，举例来说：

```typescript
class Toast {
  travel() {}
}

class Dog {
  bark() {}
}

// No explicit type guards here!
function isToast(toast: unknown) {
  return toast instanceof Toast;
}

declare let input: Toast | Dog;

if (isToast(input)) {
  input.travel(); // error before 5.5
} else {
  input.bark(); // error before 5.5
}
```

此前，TypeScript 会为 isToast 函数隐式推导一个 boolean 类型，这就导致如果你希望在 isToast 调用成立的作用域内，将 toast 类型收窄到 Toast ，就需要手动在函数返回值类型中进行类型守卫（toast is Toast）。

而现在，TypeScript 会在条件满足的情况下，为函数返回值类型提供隐式的类型守卫。

```typescript
// function isToast(toast: unknown): toast is Toast
function isToast(toast: unknown) {
  return toast instanceof Toast;
}
```

这一改进同时修正了一个此前 TypeScript 中遗留已久的问题，即数组的 filter 方法无法将数组成员类型中的 undefined / null 去除：

```typescript
declare const arr: (string | null)[];

// before 5.5: (string | null)[]
// after 5.5: string[]
const result = arr.filter((item) => item !== null);
```

此前常用的解决方式是 [ts-reset](https://www.totaltypescript.com/ts-reset) 提供的类型修正，而现在随着原生的语言支持，我们又可以少导入一行 polyfill 了。

隐式推导类型守卫需要满足以下这几个条件：

* 函数没有显式进行返回值的类型标注或类型守卫。
* 函数仅有一条 return 语句，不存在隐式 return undefined 的情况。
* 函数没有修改它的入参。
* 函数返回的是布尔类型的表达式求值，且这个表达式能够用于对参数进行类型收窄。

注意，布尔类型的表达式求值也是有一些要求的，如 `!!data` 这样基于 JavaScript 隐式转换的表达式是无法满足条件的，因为可能出现 data 是 0 的情况。

> P.S. 这个隐式转换让我想到了 ?? 与 || 的差异，我经常见到全用 `??` 或者全用 `||` 的代码，其实两种做法都不太对，毕竟有 `''` 和 `0` 这样的「有效值」，还是要按照实际场景科学使用才是。

### 索引访问的控制流分析优化 Control Flow Narrowing for Constant Indexed Accesses

现在，TypeScript 能够对索引访问进行类型收窄，前提是这里的 key 是一个常量（const 声明或函数参数）：

```typescript
declare const obj: Record<string, unknown>;
declare const key: string;

if (typeof obj[key] === 'string') {
  obj[key].toUpperCase();
}
```

这里的 「key 是常量」很关键，因为 TypeScript 并不知道你到底选择了哪个属性，只会有在 key 是个常量的前提下，才能够确保这里的类型收窄是成立的。

### 独立类型声明 Isolated Declarations

如果你了解 Isolated Modules 是怎么工作的，那理解 Isolated Declarations 其实就很简单了。Isolated Modules 常用于确保你的 TS 代码能够被 ESBuild 或 Babel 这样的编译器处理，即这些构建器并不进行类型检查，而是专注于进行语法的降级工作。也正因此，有一些特殊的语法是不能被这些编译器处理的，比如类型的重导出：

```typescript
// A.ts
import { TypeA } from './B';

export { TypeA }; // Error
```

由于Babel 编译时是单个文件进行处理的，这里的变量 TypeA 也没被标记为仅类型，所以 Babel 没法知道 TypeA 到底是个啥玩意，所以它会选择保留 A.ts 里的导出。假设 TypeA 是个类型吧，处理 B.ts 的时候它肯定会被移除掉，那 A.ts 编译后的产物运行不就报错了吗。

类似的，还有对常量枚举、命名空间等特性的禁用，都是为了避免出现其它 Transpiler 工具无法正确处理的情况。

而 Isolated Declarations 你可以理解为也是类似的作用：「为了满足某些使用场景，而对源码做出一定限制」。这里的使用场景，最重要之一的就是「并行的类型构建」。如果你有一定的 Monorepo 项目开发经验，那类型检查耗时大概率是一个折磨过你的问题。原因在于，Monorepo 中的任务顺序通常需要是按照拓扑排序执行的，即从最底层的工作区依赖开始一级级向上执行（utils→hooks→components→admin），类型检查这种必定依赖上游的任务就更不必说了。

上面我们提到了，可以通过限制使用特性来，让单文件转译工具能够正确地处理 TS 文件，那其实类型检查也可以这么安排一下——如果每个文件的导入都已经有了完整的类型声明，那岂不是可以仅仅检查这个文件，而不需要一级级溯源了？这样一来，岂不是就可以并行进行类型检查，把你的 CPU 直接榨干了。

Isolated Declarations 的限制主要包括两点：

* 启用 `declaration` 或 `composite` 。
*   对于每个文件的导出内容，必须拥有显式的类型标注，或者说能推导到一个精确的类型，示例如下：

    ```typescript
    function foo(x: unknown) {
      // Error: 推导类型是 unknown
      return typeof x === 'string' ? x.length : x;
    }

    let x: string;

    function bar() {
      // Error，x 不是一个导出的变量
      return x;
    }

    export let y: string;

    function baz() {
      // OK，y 是一个导出的变量，类型对外界可知
      return y;
    }
    ```

### TSConfig 中的模板变量

众所周知`tsconfig.json`中可以通过 extends 来复用一组公用的配置集，如继承团队规范，或在 Monorepo 中继承公用配置等 。然而，如果涉及到构建相关的配置，extends 就有点不太对劲了。举例来说：

```json
// WORKSPACE/tsconfig.base.json
{
  "compilerOptions": {
    "outDir": "dist"
  }
}
```

```json
// WORKSPACE/packages/app/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {}
}
```

你希望的效果是，在共享配置中指定输出目录为 dist，所有继承这个配置的文件仍然使用 dist 输出目录，但目录的位置应该相对于自己的 tsconfig.json，即 packages/app 下的配置。然而实际上，这里的 outDir 在解析配置时已经被处理为绝对路径，即 `WORKSPACE/dist`，因此每个继承文件必须自己再定义一次 outDir。

为了解决这个问题，5.5 版本引入了 `${configDir}` 变量来引用当前配置文件的路径，可以这么改写公共配置文件：

```json
// WORKSPACE/tsconfig.base.json
{
  "compilerOptions": {
    "outDir": "${configDir}/dist"
  }
}
```

这样一来，outDir 的路径就会基于当前继承的文件进行解析了，类似的路径配置还包括 paths、typeRoots、declarationDir 等。

### 其它

#### JSDoc 中的类型导入 Type Imports in JSDoc

TypeScript 现在支持在 JSDoc 中进行类型导入了，此前假如你想在 JSDoc 中声明一个可复用的类型别名，得要为每一个类型声明一个 `@typedef`：

```javascript
/**
 * @typedef {import("react").CSSProperties} CSSProperties
 * @typedef {import("react").HTMLAttributes} HTMLAttributes
 */

/**
 * @param {CSSProperties} styles
 */
function getStyle(styles) {}

/**
 * @param {HTMLAttributes} attrs
 */
function getAttrs(attrs) {}
```

如果从一个模块导入的类型太多，这其实是非常麻烦的，比如 remark 官方的代码全部都是使用这种方式来在 JS 中提供类型声明的，一个文件出现十几个 `@typedef` 都很正常。

而现在，你可以直接导入一组类型了：

```javascript
/** @import { CSSProperties, HTMLAttributes } from "react" */

/**
 * @param {CSSProperties} styles
 */
function getStyle(styles) {}

/**
 * @param {HTMLAttributes} attrs
 */
function getAttrs(attrs) {}
```

#### 类型移植错误

修正了 `如果没有引用 "Module"，则无法命名 "Data" 的推断类型。这很可能不可移植。需要类型注释` 错误。

这个错误常见于 pnpm 项目，原因在于 TS 在尝试为 Data 推导出类型的时候，发现这个推导类型依赖来自 Module 包的一个类型定义，但 Module 包并不在项目的直接依赖中，而是来自于符号链接的更深层的依赖，因此 TypeScript 会认为这个类型并不能直接使用。而现在 TypeScript 会检查 Module 是否真的被包括在项目的依赖树中，来判断这是否是可引用的类型。

全文完，我们 TS 5.6 见 :-)
