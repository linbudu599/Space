---
description: '2022-01-21'
---

# TypeScript 4.6 beta：递归类型检查增强、参数的控制流分析支持、索引访问的类型推导

TypeScript 已于 2022.1.21 发布 4.6 beta 版本，你可以在 [4.6 Milestone](https://github.com/microsoft/TypeScript/milestone/151) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [JavaScript and TypeScript Nightly](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第二篇 TypeScript 更新日志，上一篇是 「TypeScript 4.5 发布：新的扩展名、新语法、新的工具类型」，你可以在此账号的创作中找到。在前一篇的经验上，笔者将进一步的完善文章的描写风格，包括部分 feature 的历史背景、实际应用以及适当扩展。接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

### 上版本回顾

* TypeScript 的 4.5 版本是一个对于 NodeJS 的重要版本，引入了新的 `compilerOptions.module` 配置：`node12`以及 `nodenext`（除 node 相关以外，还有 `es2022`，主要特性是 top-level await），同时支持了 `package.json` 中的 `types` & `exports` & `imports` 解析、新的文件扩展名 `.mts` 与 `.cts` （产物对应到 `.mjs` 与 `.cjs`）
*   新的工具类型 `Awaited`，用于提取 Promise 的内部值，并替换了一批相关的 Promise 内部声明定义，如新的 `Promise.all` 定义：

    ```typescript
    interface PromiseConstructor {
      all<T extends readonly unknown[] | []>(
        values: T
      ): Promise<{ -readonly [P in keyof T]: Awaited<T[P]> }>;
    }
    ```
*   基于模板字符串类型的类型守卫：

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
      return r.type === "HttpSuccess";
    }
    ```
*   值导入与类型导入的混用：

    ```typescript
    import { Foo, type BarType } from 'lib';
    ```
* 其他一些特性，如条件类型的尾递归优化（_Tail-Recursion Elimination on Conditional Types_）、导入断言（_Import Assertions_，来自于 TC39 提案 [proposal-import-assertions](https://github.com/tc39/proposal-import-assertions)，目前处于 Stage 3）等

相比于 4.5 版本，此次的 4.6 版本带来的新特性要少一些（4.5 版本官方列出的 Major Changes 共有 13 项，4.6 版本中则是 7 项），关注的更新也重新回到了类型系统上，包括此次会被重点介绍的控制流分析增强与索引访问的类型推导。

### 支持在 super() 前执行代码（Allowing Code in Constructors Before `super()`）

我们知道 ES6 中的 Class 要求派生类的构造函数必须执行 `super()` 调用，这是为了确保 this 已被初始化完毕。TypeScript 保留了这一约束，并在此之上进一步的做了限制，在 `super()` 前的代码在某些情况下是完全不允许的，即使前面的代码中没有使用到 this。

在 TypeScript Spec 中对 Super 调用的定义是这样：

非派生类的构造函数中可以不含有 super 调用，而派生类的构造函数中则必须拥有（至少一处）super 调用，且 super 调用不允许在构造函数内部的函数、构造函数外部使用。如果以下两种情况同时满足，则对 super 语句的调用必须位于构造函数的第一行：

* 派生类
* 构造函数声明了参数属性，或类的实例属性声明带有初始化语句（initializer）

如以下这种情况：

```typescript
class Foo {
  constructor(name: string) {}
}

class Bar extends Foo {
  someProperty = true;

  constructor(name: string) {
    // Error: A 'super' call must be the first statement in the constructor when a class contains initialized properties, parameter properties, or private identifiers.(2376)
    const transformed = transformer(name);
    super(transformed);
  }
}

const transformer = (arg: string) => {
  return "linbudu";
};
```

这实际上也是一个常见的场景，即在调用 super 前需要对参数做校验/转换，如果是简单的操作还好，我们可以直接 `super(transformer(args))` 来绕过校验，但如果想要将各个步骤单独的独立开，难道要 `super(transformer(validator(foo(args))))`？对于没有使用 this 的代码来说，其实在 super 前调用时不应该抛出错误（实际上 ES6 就是支持这么做的）。

简单地说，这一特性允许了在 super 调用前去执行没有引用 this 的代码，对于强依赖 OOP 的类库，它们的构造函数可以更整洁一些了。

### 递归类型检查增强 Improved Recursion Depth Checks

我们知道 TypeScript 的类型系统属于“鸭子类型”，即结构化的类型系统（Structural Type System），这也就意味着以下代码是成立的：

```typescript
interface Source {
  prop: "foo" | "bar";
}

interface Target {
  prop: "foo" | "bar";
}

function check(source: Source, target: Target) {
  target = source;
}
```

在这里将 Source 结构赋值给 Target 结构的类型体是允许的，因为**结构化类型系统通过比较这两个结构内部的属性是否可以具有兼容性（或者说是否可分配），来判断这两个结构的兼容性**，如果我们将这里作为被赋值类型的 Target 的结构中属性（`prop`）进行修改，使得其与 `Source.prop` 之间不再可分配，则赋值操作会抛出一个错误：

```typescript
interface Source {
  prop: "foo" | "bar";
}

interface Target {
  prop: "foo";
}

function check(source: Source, target: Target) {
  // error!
  // Type 'Source' is not assignable to type 'Target'.
  //   Types of property 'prop' are incompatible.
  //     Type 'foo' | 'bar is not assignable to type 'foo'.
  //        Type 'bar is not assignable to type 'foo'.
  target = source;
}
```

在这里我们直接使用直观的字面量类型组成的联合类型作为示例，因此整体看起来非常直观，那么，如果我们引入泛型呢？

```typescript
interface Source<T> {
  prop: Source<Source<T>>;
}

interface Target<T> {
  prop: Target<Target<T>>;
}

function check(source: Source<"foo" | "bar">, target: Target<"foo" | "bar">) {
  target = source;
}
```

你现在还能直观的判断出这里是否是允许的吗？我们继续按照结构类型系统的比较方式，去比较这里两个结构体的 prop，即 `Source<Source<T>>` 和 `Target<Target<T>>`，然后你就会发现这是个无限循环娃套娃的过程。

TypeScript 当前对这种情况的处理逻辑是执行一定深度的展开检查，如果还没完事就判定这是一个无限循环，则认为两个类型是兼容的，称为**启发式的递归类型检查**。

通常情况下这已经足够了，但还存在着一些漏网之鱼：

```typescript
interface Foo<T> {
  prop: T;
}

declare let x: Foo<Foo<Foo<Foo<Foo<Foo<string>>>>>>;
declare let y: Foo<Foo<Foo<Foo<Foo<string>>>>>;

x = y;
```

我们能够很明显的确定，这里的赋值操作应当是不成立的，因为 y 的嵌套要少了一层，但是先前版本的 TypeScript 不会报错。对于之前的递归类型检查来说，它关注的是这种嵌套结构展开的情况（无论是前面的例子中，结构的属性中引用了结构来进行套娃，还是这一例子中，结构的泛型参数中引用了结构进行套娃），而不是实际声明的情况，即在进行一定深度检查后直接认为两个类型兼容。

> 此特性在 4.5.3 版本就已经引入，因此以上这种情况在 TypeScript 4.5.3 以后的版本就能够检查出错误。

而 4.6 版本进一步增强了递归类型检查，能够区分这里的两种情况，即不再关注 **结构的泛型参数中引用了结构进行套娃** 这种来自于明确指定的特殊情况，对于这种情况中的无限嵌套也能够更迅速的判断出来。明显体现了其效果的则是一些 DefinitelyTyped Package 的类型检查工作减少了 50% 以上的成本，如 @types/yup 以及 @types/redux-immutable 等。如果你对具体实现感兴趣，可以参考 [#46599](https://github.com/microsoft/TypeScript/pull/46599)。

### 索引访问的类型推导 Indexed Access Inference Improvements

索引访问类型（Indexed Access Types）使得你可以通过接口的键名获取键值代表的类型，如：

```typescript
type Person = { age: number; name: string; alive: boolean };
// number
type Age = Person["age"];

// number | string
type NameOrAge = Person["age" | "name"];

// number | string | boolean
type All = Person[keyof Person];
```

一些“索引”相关的概念可能造成混淆，如：

* 索引类型
* 索引签名类型
* 索引访问
* keyof
* 映射类型

这些概念我在先前的文章 「TypeScript 的另一面：类型编程」（同样见此账号）中有详细的介绍，这里仅做简单的区分。

* 索引类型：索引类型不是一种具体的类型（就像映射类型那样），而是一系列基于索引的类型操作的统称，包括**索引签名类型**、**索引访问**等。
*   索引签名类型：用于快速建立一个内部字段类型一致的接口，或为接口的未知属性提供访问支持：

    ```typescript
    // 等价于 Record<string, string>
    interface Foo {
      [keys: string]: string;
    }

    // 避免访问 Bar['job'] 时出错，常见于重构场景或动态性强的接口结构
    interface Bar {
      name: string;
      age: number;
      [keys: string]: unknown;
    }

    type Job = Bar['job'] as AcutallyJobType
    ```
*   索引访问，即通过索引来访问接口的类型（见上面的例子），或通过索引访问数组、元组：

    ```typescript
    const stringArr = <const>["lin", "bu", "du"];

    // "lin" | "bu" | "du"，如果移除 as const 声明，则为 string
    type TypeFromArr = typeof stringArr[number];

    const tuple = <const>["linbudu", true, 18];

    // true | "linbudu" | 18 ，如果移除 as const 声明，则为 boolean | string | number
    type TypeFromTuple = typeof tuple[number];
    ```

    从这里数组的索引访问，`typeof tuple[number]` 我们可以发现**一个值得关注的点是，这里的索引并不是“值”，而是索引的类型（即字面量类型）**，所以 `Person['age']` 实际上等价于：

    ```typescript
    type AgeLiteralType = "age";

    type AgeType = Person[AgeLiteralType];
    ```
*   keyof 与 映射类型：`keyof` 和 `typeof` 一样，属于一个独立的类型操作符，**不属于索引类型**。它的作用是获取一个结构下所有的 key 的字面量类型组成的联合类型，如：

    ```typescript
    // "name" | "age" | "alive"
    type PersonProps = keyof Person;
    ```

    **只有在和映射类型一同使用时，它才能访问到 key 对应的类型。**

    映射类型类似于 JavaScript 数组的 map 方法，它通常被用在工具类型中基于已有接口做各种操作，如我们将一个接口的所有类型映射到 string：

    ```typescript
    interface A {
      a: boolean;
      b: string;
      c: number;
      d: () => void;
    }

    type StringifyA<T> = {
      [K in keyof T]: string;
    };
    ```

    **需要注意的是，映射类型与 keyof 是彼此独立的，只是搭配有奇效。**

    更常见的场景是与`T[k]` 这种形式的索引类型访问一起使用：

    ```typescript
    type Clone<T> = {
      [K in keyof T]: T[K];
    };
    ```

这些相关的概念可能带来一些混淆，但认真的对待的话，它们之间还是有非常明显差异的。

再说回 4.6 版本中的索引访问类型推导，在此前索引访问类型实际上已经有了一定的类型推导能力，但还存在着许多不足，看以下的例子：

```typescript
type UnionRecord =
  | { kind: "n"; v: number; f: (v: number) => void }
  | { kind: "s"; v: string; f: (v: string) => void }
  | { kind: "b"; v: boolean; f: (v: boolean) => void };

type VTypes = UnionRecord["v"];
```

这里 `VTypes` 能够被正确的推导为 `string | number | boolean`，但这一推导结果在以下就将导致一个错误：

```typescript
function processRecord(rec: UnionRecord) {
  rec.f(rec.v); // Error, 'string | number | boolean' not assignable to 'never'
}
```

由于参数 `rec` 的类型只会是联合类型的一个分支，所以其中的 v、f 类型应当是对应的，但这里 v 仍然是联合类型，而 f 则变成了 `(v: never) => void`（never 来自于 `'string | number | boolean'` 的并集，很明显它们并没有并集）。这个检查结果意味着检查的执行是在检查每一条记录中的 v 是否可分配给每一条记录的 f 作为参数，但这并不符合我们的预期。

在 4.6 版本以前实际上我们也能够解决这一问题，这里不符预期的结果很明显是来自于联合类型的各个类型分支没有隔离，我们可以**通过泛型来显式的纠正类型系统的控制流分析**，这里一次只会有一个类型分支：

```typescript
type RecordTypeMap = { n: number; s: string; b: boolean };

type UnionRecord<K extends keyof RecordTypeMap = keyof RecordTypeMap> = {
  [P in K]: {
    kind: P;
    v: RecordTypeMap[P];
    f: (v: RecordTypeMap[P]) => void;
  };
}[K];

function processRecord<K extends keyof RecordTypeMap>(rec: UnionRecord<K>) {
  rec.f(rec.v); // Ok
}
```

很明显，当 kind 为 n，则 v 一定为 `number`，f 一定为 `(v: number) => void`，因此不会再抛出前面的错误。

此特性的引入即是让我们无需再使用上面这些代码，也能够在索引类型映射后的类型中获得精确地类型提示，即按分支归类。

### 参数类型的控制流分析支持 Control Flow Analysis for Dependent Parameters

在写这篇文章的时候我还同时写了一篇 TypeScript 控制流分析的相关文章，如果你对控制流分析这一概念并不了解，推荐直接阅读那一篇文章， 「TypeScript 中的控制流分析演进：以 4.6 版本新特性为引」，在这里，我们只做简单的介绍。

这一特性实际上类似于在 4.5 版本中引入的基于模板字符串类型的类型守卫，其实际上也就是**模板字符串类型的控制流分析**支持：

```typescript
export interface Success {
  type: `${string}Success`;
  body: string;
}

export interface Error {
  type: `${string}Error`;
  message: string;
}

function response(r: Success | Error) {
  if (r.type === "HttpSuccess") {
    return r.body;
  }
  if (r.type === "HttpError") {
    return r.message;
  }

  return null;
}
```

而对比的说，参数类型的控制流分析支持实际上就是支持了**参数类型的可辨识联合类型推导**。

来了个新概念，**可辨识联合类型（Discriminated Unions 或 Tagged Unions）**，实际上它没有什么复杂的，我们上面已经出现了示例：

```typescript
type UnionRecord =
  | { kind: "n"; v: number; f: (v: number) => void }
  | { kind: "s"; v: string; f: (v: string) => void }
  | { kind: "b"; v: boolean; f: (v: boolean) => void };
```

首先我们知道，可辨识联合类型必然是联合类型，那么要如何能让它进化成可辨识的？实际上只需要有一个属性存在于每一个类型分支，且在每个分支中的类型都不相同（可以是基本类型，高级类型，字面量类型等，只需要不同即可，但一般推荐使用字面量类型），称这个用于区分的属性为**可辨识属性（Discriminant Property 或 Tagged Property）**。

其次，通常在使用联合类型时，我们需要先\*\*收窄（Narrowing）\*\*到某一个类型分支，最好的方式是通过可辨识联合属性的判断，如其字面量类型的判断（`if (type === 'success')`）、 `typeof` 判断（`if (typeof obj.kind === 'string')`）等。

而这一特性关注的就是参数类型（我们直接使用元组描述）的辨识，如：

```typescript
type Args = ["a", number] | ["b", string];

const f1: Func = (kind, payload) => {};
```

在上面这种情况中，通过 kind 的判断将联合类型缩窄到对应的部分，其使用方式和常见的类型守卫一致：

```typescript
const f1: Func = (kind, payload) => {
  if (kind === "a") {
    payload.toFixed(); // 'payload' narrowed to 'number'
  }
  if (kind === "b") {
    payload.toUpperCase(); // 'payload' narrowed to 'string'
  }
};
```

在第一个 if 块中，由于我们进行了对应的联合类型辨识，payload 的类型能够被收窄到 number，而这背后的分析过程即是控制流分析，详见另一篇文章。

### 新的性能分析工具 TypeScript Trace Analyzer

TypeScript Compiler 提供了 `--generateTrace` 选项来生成 compiler 在本次编译工作中的耗时占比，并通过 Edge/Chrome 支持 的 tracing 功能来可视化的展示报告（参考 [Performance Tracing](https://github.com/microsoft/TypeScript/wiki/Performance#performance-tracing)），如使用 Chrome Tracing：

![](https://img.alicdn.com/imgextra/i2/O1CN016nZyhX1eWd4hUpZye\_!!6000000003879-2-tps-2272-646.png)

而本次 [Trace Analyzer](https://github.com/microsoft/typescript-analyze-trace) 的发布则主要用于更直观、更清晰的展示报告（没能成功跑起来，就先不放图了）。

### 破坏性变更

#### 移除对象解构中的非通用类型

在对一个通用对象（如类）使用解构赋值与 rest 展开操作符时，现在对象中的方法以及具有 getter、setter 的这一类不可解构的值将从 rest 的类型中被剔除：

```typescript
class Thing {
  someProperty = 42;

  someMethod() {
    // ...
  }
}

function foo<T extends Thing>(x: T) {
  let { someProperty, ...rest } = x;

  rest.someMethod();
}
```

在这里，rest 的类型将被正确的推导为 `Omit<T, "someProperty" | "someMethod">`（在此前则是 `Omit<T, "someProperty">`，但实际上 `someMethod` 是不可 spread 的，即可以 `x.someMethod`，但不能 `rest.someMethod`）。

### 对 JavaScript 文件的语法检查

> 这实际上也是 4.6 版本的主要新特性之一（More Syntax and Binding Errors in JavaScript）

现在，对 TypeScript 中被 include 的 JavaScript 文件，其语法错误也将被提示出来，如重复声明、对 export 声明使用了修饰符、在 switch case 语句出现多次的 default case 等。

类似于 TypeScript 文件，你也可以通过 `@ts-nocheck` 来禁用对此文件的类型检查。

### 总结

本次 TypeScript 4.6 beta 版本的主要新特性就介绍到这里，由于 TypeScript 团队的严谨性，通常 beta 版本和正式版本只会有非常小的微调，所以你基本上可以认为这就是 TypeScript 4.6 正式版本的主要特性。而在发布间隔上，按照以往的经验，通常 beta 版本和正式版本的发布会间隔一个半月左右。

全文完，我们 TypeScript 4.7 见。
