# TypeScript 5.4 beta: NoInfer 类型、闭包类型分析优化、条件类型判断优化等

TypeScript 已于 2024.1.23 发布 5.4 beta 版本，你可以在 [5.4 Iteration Plan](https://github.com/microsoft/TypeScript/issues/56948) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) ，并选择为项目使用 VS Code 的 TypeScript 版本，来更新内置的 TypeScript 支持：

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

本篇是笔者的第十篇 TypeScript 更新日志，上一篇是 「TypeScript 5.3 beta：Import Attributes 提案、Throw 表达式、类型收窄优化」，你可以在此账号的创作中找到（或在掘金/知乎/Twitter搜索**林不渡**)，接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

### NoInfer 工具类型

如果你经常使用 TypeScript 的泛型，可能会发现一个不那么符合直觉的地方：在函数签名中，如果有多个参数类型引用同一个泛型参数，那么其表现行为会是，这个泛型参数被推导为一个能够尽可能满足所有参数类型的类型（即在类型层级上表示为所有参数类型的公共父类型）。

来看实际的例子：

```typescript
function checkLevel<T extends string>(levels: T[], defaultLevel?: T) {}

// 泛型参数推导：checkLevel<"easy" | "normal" | "hard">
checkLevel(['easy', 'normal', 'hard'], 'easy');

// 泛型参数推导：checkLevel<"easy" | "normal" | "hard" | "hell">
checkLevel(['easy', 'normal', 'hard'], 'hell');
```

我们预期的效果是，从第一个参数推导出所有可用的 levels 的联合类型，然后第二个默认值的类型也应当来自于这个联合类型才对，事实却是 TypeScript 把第二个参数推导出的类型也塞进了泛型类型里。

这个场景下，字面量类型的公共父类型直接就是联合类型，而另外一种常见的场景则是存在显式继承关系的类型：

```typescript
class Animal {
  move;
}
class Dog extends Animal {
  woof;
}

function doSomething<T>(value: T, getDefault: () => T) {}

// 泛型推导为 doSomething<Animal>;
doSomething(new Dog(), () => new Animal());
```

要解决这里的两个推导问题其实也很简单，只需要为第二个参数多声明一个泛型，然后使这个泛型约束到第一个泛型即可：

```typescript
function checkLevel<T extends string, K extends T>(
  level: T[],
  defaultLevel?: K
) {}

// 类型“"hell"”的参数不能赋给类型“"easy" | "normal" | "hard" | undefined”的参数。
checkLevel(['easy', 'normal', 'hard'], 'hell');
```

看起来没问题，但实际上这个新的泛型参数 K 大概率是一个无意义的泛型参数——如何判断一个泛型参数是否有意义？看它有没有被消费就对了，这里 checkLevel 的返回值类型应当会依赖 T ，但不会依赖 K：

```typescript
function checkLevel<T extends string, K extends T>(
  level: T[],
  defaultLevel?: K
): T { }
```

如果只是为了类型约束而单独声明一个泛型参数，其实并不是很好的实践，因此 TypeScript 5.4 版本引入了内置工具类型 NoInfer (intrinsic)，用于在这种情况下阻止泛型参数的推断：

```typescript
function checkLevel<T extends string>(level: T[], defaultLevel?: NoInfer<T>) {}

// 类型“"hell"”的参数不能赋给类型“"easy" | "normal" | "hard" | undefined”的参数。
checkLevel(['easy', 'normal', 'hard'], 'hell');
```

在这种情况下，TypeScript 不会再将这个参数的类型合并到泛型中，而是会使用泛型参数已获得的类型，来对这个参数进行类型检查。

作为一个工具类型，NoInfer 还可以在条件类型语句中使用。先看不使用 NoInfer 的例子：

```typescript
type Foo<T> = T extends { a: infer U; b: infer U } ? U : never;

type Foo1 = Foo<{ a: string; b: string }>; // string
type Foo2 = Foo<{ a: string; b: number }>; // string | number
```

和函数中一样，infer 类型 U 会被推导为所有引用位置的联合类型，现在来加一个 NoInfer 试试：

```typescript
type Bar<T> = T extends { a: infer U; b: NoInfer<infer U> } ? U : never;

type Bar1 = Bar<{ a: string; b: string }>; // string
type Bar2 = Bar<{ a: string; b: number }>;
```

第一眼看上去，这个时候 Bar2 的类型也是 string 才对，因为 NoInfer 只阻止了第二处的泛型推导，并没有阻止第一处。但实际上 Bar2 的类型是 never，我个人理解是，在这种情况下 NoInfer 会导致条件类型的条件不成立，模式匹配根本就没有发生，直接走了 false 的 never 类型。

### 闭包类型 CFA 优化

> 此前这个系列的文章使用过很多不同的方式来描述「类型收窄」这个行为，比如类型分析、类型收窄、类型推断等等，后续为了统一概念，将全部归纳为「类型控制流分析」（的优化），并简称为 TCFA（Typing Control Flow Analysis）

两年前我在 [TypeScript 中的类型控制流分析演进](https://linbudu.gitbook.io/spaces/typescript-in-deep/typescript-zhong-de-lei-xing-kong-zhi-liu-fen-xi-yan-jin) 分析过 TypeScript 从 2.0 版本到 4.6 版本一路下来的 TCFA 优化，你可以很明显感觉到，TypeScript 正在变得越来越聪明。在上一个版本中 TypeScript 优化了 `switch(true)` 、使用布尔值比较的类型守卫下的 TCFA 表现：

```ts
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

而在 5.4 版本，TypeScript 对闭包下的 TCFA进行了优化。

先来看一个简单的例子：

```typescript
function f1() {
  let x: string | number;

  x = 'abc';
  console.log(x); // string

  x = 42;
  console.log(x); // number
}
```

在为具有联合类型的变量赋值后，TypeScript 能够分析出这个变量接下来的类型（直到下一次赋值），但如果赋值后不是在当前的作用域内，而是在一个闭包内访问，此前 TypeScript 是无法分析出类型的，因为你无法确定 action 是在什么时候调用回调函数的。

```typescript
declare function action(cb: () => void): void;

function f1() {
  let x: string | number;

  x = 'abc';
  action(() => {
    console.log(x); // string | number
  });

  x = 42;
  action(() => {
    console.log(x); // string | number
  });
}
```

TypeScript 5.4 版本为这个场景下的 TCFA 进行了优化，现在它能够正确分析出这个变量最后一次赋值以后的类型——此时变量的类型不会再发生改变了，即闭包捕获的值已经固定，在上面的例子中就是这样的：

```typescript
declare function action(cb: () => void): void;

function f1() {
  let x: string | number;

  x = 'abc';
  action(() => {
    console.log(x); // string | number
  });

  x = 42;
  action(() => {
    console.log(x); // number
  });
}
```

但需要注意的是，如果这个变量在另外一个嵌套的函数中有赋值语句，那么这个分析就失效了：

```typescript
declare function action(cb: () => void): void;

function f1() {
  let x: string | number;

  x = 'abc';

  setTimeout(() => {
    x = 42;
  }, 500);

  action(() => {
    console.log(x); // string | number
  });
}
```

这是因为，TypeScript 并不能知道这个赋值的嵌套函数，是在什么时候调用的，因此也就无法确定真正的最后一次赋值。

看一个更接地气的例子：

```typescript
function appendUrlParams(url: string | URL, params: Record<string, number>) {
  if (typeof url === 'string') {
    url = new URL(url);
  }

  Object.entries(params).forEach(([param, value]) => {
    // Property 'searchParams' does not exist on type 'string | URL'. error before 5.4, now ok.
    url.searchParams.set(param, value.toString());
  });

  return url.toString();
}
```

在 5.4 版本，TypeScript 现在能够分析出 `Object.entries(params).forEach` 中使用的 url 一定是 URL 类型。

这一优化实际上对所有作用域捕获都会生效——除了会享有作用域提升的函数声明、类声明以外：

```typescript
function f2() {
  let x: string | number;
  x = 42;
  let a = () => {
    x; /* number */
  };
  function g() {
    x; /* string | number */
  }
}
```

当然，类型收窄并不仅仅对联合类型生效，隐式具有 any 类型的变量同样能够生效：

```typescript
declare function action(cb: () => void): void;

function f3() {
  let x;

  x = 'abc';
  action(() => {
    x; // any
  });

  x = 42;
  action(() => {
    x; // number
  });
}
```

### Throw Expression

此特性是对 TC39 提案 [proposal-throw-expression](https://github.com/tc39/proposal-throw-expressions) 的支持，其目前处于 stage 2 阶段，由 TypeScript 团队的 [Ron Buckton](https://github.com/rbuckton) 提出。实际上，在 5.3 Iteration Plan 中就已经包括此特性，但其没能够在 5.3 版本正式发布，在这里出现是因为它将在 5.4 版本中发布吗？也不是 :-)，和 5.2 版本一样，它出现在了 Iteration Plan 中，但没有出现在 DevBlog 中，但谁让我写都写了呢。

Throw Expression 提案允许你像使用表达式一样来使用一个 throw 语句，举例来说，此前我们写一个对黑名单用户抛出错误的逻辑可能是这样的：

```javascript
const safeUser = isSafeUser();

if(!safeUser){
  throw new Error('...')
}
```

我们需要将 throw 语句写在单独的代码块里才能正常执行，而现在使用 Throw Expression，你可以直接这么写：

```javascript
isSafeUser() ? void 0 : throw new Error('...');
```

你也可以使用它来进行“赋值”：

```javascript
const user = isSafeUser() || throw new Error('...');
```

可以这么理解，Throw Expression 并不是真的把这个错误赋值给了一个变量，而是当你的代码执行到这个 Throw Expression 时，会执行这个 throw 语句而已。可以把它应用在各种默认值场景下：

```javascript
function readFileSync(path = throw new PathNotProvidedError()) { };

function getEncoder(encoding) {
  const encoder = encoding === "utf8" ? new UTF8Encoder()
    : encoding === "utf16le" ? new UTF16Encoder(false)
      : encoding === "utf16be" ? new UTF16Encoder(true)
        : throw new Error("Unsupported encoding");
};
```

Throw Expression 目前在 Babel 中被实现为一元表达式节点，即 UnaryExpression，就像 `const visitor = !userLogin` 中的 `!userLogin` 一样。UnaryExpression 的核心节点包括 operator 与 argument，在这里分别是 `!` 与 `throw`，`userLogin` 与 `new Error()`。

另外，Throw Expression 编译的降级产物实际上是一个 IIFE ，比如上面的例子的编译结果大致是这样的：

```javascript
function readFileSync(path = function (e) {
  throw e;
}(new PathNotProvidedError())) {}
;

function getEncoder(encoding) {
  const encoder = encoding === "utf8" ? new UTF8Encoder() : encoding === "utf16le" ? new UTF16Encoder(false) : encoding === "utf16be" ? new UTF16Encoder(true) : function (e) {
    throw e;
  }(new Error("Unsupported encoding"));
}
;
```

> 你也可以在 [Babel Playground](https://babeljs.io/repl#?browsers=defaults%2C%20not%20ie%2011%2C%20not%20ie\_mob%2011\&build=\&builtIns=false\&corejs=3.21\&spec=false\&loose=false\&code\_lz=GYVwdgxgLglg9mABAJwKYEMAmAxGAbVAZQE9IAKAB3SgAtEBeRW5OAd0TFXYAVqaA5OFG4sAbjEypMAUWQtkZAJSLEAb0QBfANwAoHaEiwEiAOaoo0yHEkLUVzDDAmVqnYkQQEAZyiI7nmwY\_e0cTBnpGACIQKGAADkjEAH4OLkQAVQAVbDjLANQFRTd3RAAuYIDQ8KiY4ABGADYCRJTOdizsRrzrArJgdDwvVCKSkvL\_ayqImtjGgCNUFtT27K77XqhkEGHi0fdy5jZlxFl5Mkj0sC8QCgo4ZCgpCsmnSMVdbSA\&debug=false\&forceAllTransforms=false\&modules=false\&shippedProposals=false\&circleciRepo=\&evaluate=false\&fileSize=false\&timeTravel=false\&sourceType=module\&lineWrap=true\&presets=env%2Creact%2Cstage-2\&prettier=false\&targets=\&version=7.23.8\&externalPlugins=\&assumptions=%7B%7D) 自行玩耍\~

### Object.groupBy & Map.groupBy

TypeScript 5.4 版本新增了 `Object.groupBy` 与 `Map.groupBy` 方法的类型声明，这两个方法来自于 [proposal-array-grouping](https://github.com/tc39/proposal-array-grouping) 提案，其已进入 Stage 4，将成为 ECMAScript 的一部分。

这两个方法其实类似于 Lodash 中的 groupBy，但不同点在于，`Object.groupBy` 与 `Map.groupBy` 分别会将结果存储为 Object 与 Map 的形式：

```typescript
const array = [1, 2, 3, 4, 5];

Object.groupBy(array, (num, index) => {
  return num % 2 === 0 ? 'even': 'odd';
});
// =>  { odd: [1, 3, 5], even: [2, 4] }

// using an object key.
const odd  = { odd: true };
const even = { even: true };
Map.groupBy(array, (num, index) => {
  return num % 2 === 0 ? even: odd;
});
// =>  Map { {odd: true}: [1, 3, 5], {even: true}: [2, 4] }
```

### 条件类型判断优化

```typescript
type IsArray<T> = T extends any[] ? true : false;

function f1<U extends object>(x: IsArray<U>) {
    let t: true = x;   // Error: Type 'IsArray<U>' is not assignable to type 'true'.
    let f: false = x;  // No Error
}
```

在这个例子的函数 f1 内部，由于此时暂时没有足够的类型信息，无法知晓 U 可能的类型，TypeScript 会使用 U 的约束 object 来进行类型分析，而 `object extends any[]` 并不成立，因此上面的例子里此前 TypeScript 分析出的 x 的类型是 false 字面量类型。

但实际上，只使用约束来判断条件类型是不那么准确的，比如，U 可以是 `any[]` 类型，`any[] extends any[]` 成立，此时 IsArray 的返回值是 true，也可以是 `{}` 类型，`{} extends any[]` 不成立，此时返回值是 false，有点像条件类型中的判断条件使用 any extends 一样，此时它包含了让条件成立的一部分，以及让条件不成立的一部分，最终结果是条件类型两个分支组成的联合类型：

```typescript
type Result1 = any extends 'linbudu' ? 1 : 2; // 1 | 2
type Result2 = any extends string ? 1 : 2; // 1 | 2
type Result3 = any extends {} ? 1 : 2; // 1 | 2
type Result4 = any extends never ? 1 : 2; // 1 | 2
```

在 5.4 版本，TypeScript 修正了这一判断行为，现在它能够分析出这种「可能成立也可能不成立」的情况了，因此在一开始的例子里 x 会被推导为 boolean 类型。同时，对于必定成立、必定不成立的类型推导仍然能够正常工作：

```typescript
type IsArray<T> = T extends number ? true : false;

function f1<U extends string>(x: IsArray<U>) {
  let t: true = x; // 不能将类型“IsArray<U>”分配给类型“true”。不能将类型“false”分配给类型“true”。
  let f: false = x;
}

function f2<U extends number>(x: IsArray<U>) {
  let t: true = x; // 不能将类型“IsArray<U>”分配给类型“false”。 不能将类型“true”分配给类型“false”。
  let f: false = x; // Error, but previously wasn't
}
```

### 其它变更

#### 即将移除的配置

TypeScript 的废弃策略是 5 个 minor 版本的渐进式，如 `compilerOptions.out` 将在 5.5 版本中被移除，那么从 5.0 版本开始，只要使用了此配置就会得到一个错误，需要显式设置 `ignoreDeprecations: "5.0"` 。

类似的，`suppressImplicitAnyIndexErrors`、`noStrictGenericChecks` 等配置将在 5.5 版本中被正式移除，可以参考 [Upcoming Changes from TypeScript 5.0 Deprecations](https://devblogs.microsoft.com/typescript/announcing-typescript-5-4-beta/#breaking-changes) 来了解所有相关配置。

本次的 Devblog 解析就到这里了，其实还有一些如 [Support for `require()` calls in `--moduleResolution bundler` and `--module preserve`](https://github.com/microsoft/TypeScript/pull/56785)、[Isolated Declarations](https://github.com/microsoft/TypeScript/issues/47947) 这样的功能没有介绍到，原因则是因为它们的牵涉较广，需要大量的前置知识铺垫，同时也不太会被业务开发者直接感知到，这里就先跳过了，如果你有兴趣，请关注笔者的专栏，后续可能会在专栏更新世界观级别的介绍也说不定（画饼）。

全文完，我们 TypeScript 5.5 beta 见 :-)
