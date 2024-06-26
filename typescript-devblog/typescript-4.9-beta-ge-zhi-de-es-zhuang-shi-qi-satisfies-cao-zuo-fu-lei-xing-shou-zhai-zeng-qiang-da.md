---
description: '2022-09-26'
---

# TypeScript 4.9 beta：鸽置的 ES 装饰器、satisfies 操作符、类型收窄增强、单文件级别配置等

TypeScript 已于 2022.09.23 发布 4.9 beta 版本，你可以在 [4.9 Iteration Plan](https://github.com/microsoft/TypeScript/issues/50457) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](http://link.zhihu.com/?target=https%3A//marketplace.visualstudio.com/items%3FitemName%3Dms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第五篇 TypeScript 更新日志，上一篇是 「TypeScript 4.8 beta 发布：正在路上的装饰器、类型收窄增强、模板字符串类型中的 infer」，你可以在此账号的创作中找到（或在掘金/知乎搜索**林不渡**），接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

#### 鸽置的 ECMAScript 装饰器

在上一篇的 4.8 beta 中我们曾经说过，对齐 ECMAScript 的新版 TS 装饰器（相对于现在的 Experimental Decorators）实现正在路上，但在 4.9 beta 的日志中，Daniel Rosenwasser 表示由于新版装饰器提案中还存在一些需要讨论的细节，因此预计要在下一个版本，即 5.0 版本中才会得到实现。

但已经可以确定的是，`--experimentalDecorators` 与 `--emitDecoratorMetadata` 这两个配置仍然会被保留，用于启用旧版装饰器，而新版装饰器无需配置即会默认支持。

> 非常合理，这么重磅的特性当然要版本凑个整才有仪式感。

在这里，我们就直接引用上一篇文章中的相关说明，只作稍微修改。

在 2022 年 4 月份的 TC39 双月会议上，装饰器提案成功进入到 Stage 3，这也是装饰器提案历经数个版本以后离 Stage 4 最近的一次。TypeScript 中同样大量使用了装饰器相关的语法，但实际上 TS 中的装饰器（experimental）、Babel 中的装饰器（legacy）都是基于第一版的装饰器提案进行实现的，而目前终于到达 Stage 3 的装饰器提案已经是第三版了。

> 如果你有兴趣了解更多装饰器的历史，可以阅读笔者的 [走近 MidwayJS：初识 TS 装饰器与 IoC 机制](https://juejin.cn/post/6859314697204662279) 中的介绍，或者贺师俊（Hax）老师在 [是否应该在 production 里使用 typescript 的 decorator？](https://link.juejin.cn/?target=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F404724504) 的回答。

随着新版装饰器提案的更新，TypeScript 势必也需要对应地进行支持，但由于其工作量较大，8 月份的 4.8 正式版本与预计在 11 月发布的 4.9 正式版本中都不包含新版装饰器的实现，我们预计要在 2023 年过年时才能在 5.0 beta 版本中一睹它的真容。但无论如何，其最终实现必然是遵循装饰器提案 [proposal-decorators](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Ftc39%2Fproposal-decorators) 的，因此你可以阅读笔者此前发表的 [ECMAScript 双月报告：装饰器提案进入 Stage 3](https://link.juejin.cn/?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F6PTcjJQTED3WpJH8ToXInw) ，了解新版装饰器的功能、旧版装饰器的废弃原因，以及新版装饰器如何不通过反射元数据的方式实现依赖注入。

虽然我们迎来了新版装饰器，但也无需担心旧版装饰器从此就被扫进历史的尘埃里了，对旧版装饰器的支持肯定是会被保留相当长一段时间的，语言支持、框架改进、用户接受，每一步都快不起来。我们可能要到 TypeScript 20.0 beta 版本中才会看到官方宣布将废弃对实验性装饰器的支持，希望那时笔者仍然在更新此专栏。当然，如果 Angular、Nest、InversifyJs 这些框架和工具库能快速跟进，那么这一进程可能会大大加快。

对于使用者来说，基本不用担心额外的学习成本，新版装饰器在大部分情况下能完全覆盖掉旧版装饰器的能力。但对于框架基础库开发者来说，两个版本装饰器之间的差异确实相当大，如装饰器的运行顺序，以及元数据相关等。

#### satisfies 操作符

这是 4.9 版本中最重要的新特性（没有之一），这一特性引入了新的操作符 satisfies ，我们能够使用它来进行更安全地向上断言，即 upcast。

是不是一上来信息量有点大？我们从日常的使用说起好了。

首先，TypeScript 中我们并不需要为每个变量提供精确地类型标注，其强大的推导能力能够自动地完成某些类型信息的推导。如以下这个对象：

```typescript
// 一块神奇调色板
const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
};
```

其类型将被自动推导为：

```typescript
interface Palette {
  red: number[];
  green: string;
  blue: number[];
}
```

在后续使用这个对象时，也将直接使用这份推导出来的类型信息：

```typescript
palette.green.startsWith('#'); // √ boolean
palette.red.startsWith('#'); // × 类型“number[]”上不存在属性“startsWith”。
```

但在这种时候，由于我们是从值推导得到类型，而不是使用类型约束值，那么在我们提供了错误的值时，推导得到的类型信息也将出现问题，比如在这里我不小心打错了字：

```typescript
const palette = {
  red: [255, 0, 0],
  // typo
  grren: '#00ff00',
  blue: [0, 0, 255],
};
```

这样推导出来的类型信息就有问题了，而为了避免这种问题，我们通常会使用显式地类型标注：

```typescript
type Colors = 'red' | 'green' | 'blue';
type RGB = [number, number, number];

const palette: Record<Colors, string | RGB> = {
  red: [255, 0, 0],
  // × 不存在此属性
  grren: '#00ff00',
  blue: [0, 0, 255],
};
```

看起来我们通过工具类型 + 字面量联合类型 + 元组类型完成了非常精确的类型标注，但现在又出现了新的问题，我们的调用出现了类型错误：

```typescript
palette.green.startsWith('#'); // × 类型“string | RGB”上不存在属性“startsWith”
palette.red.startsWith('#'); // × 类型“string | RGB”上不存在属性“startsWith”
```

这是因为，我们使用的类型描述实际上是在将每个属性的键值类型都确定为一个联合类型——但这实际上是不符合的！

你可能会想，只要用一个精确的接口类型，规定每个颜色应该是什么类型不就好了？那，如果这个神奇调色盘有上百种颜色呢？

要解决这个问题，我们需要先了解类型标注的本质。

在我们进行变量类型信息地标注时，其实是在告诉 TypeScript 类型系统，这个变量的值必须完全符合这个类型，在后续使用这个变量时其类型信息会完全使用我们提供地类型信息，而不是其推导出的类型信息，从这个角度看，其实类型断言也是类似的功能。

也就是说，我们现在是**让值完全符合类型，然后使用我们提供的类型信息**。而我们实际需要的效果则是，**让值符合类型的前提下，结合使用值推导出的类型信息**，也就是说 palette 只需要满足类型约束，其键值类型不会使用 `string | RGB` ，而是仍然使用每个属性访问推导出的对应类型。

说了这么多，其实就是 4.9 版本引入了新的操作符 satisfies ，来支持了这个能力啦：

```typescript
const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
} satisfies Record<Colors, string | RGB>;

// string
palette.green.startsWith('#'); // √
// [number, number, number]
palette.red.find(); // √
// [number, number, number];
palette.blue.entries(); // √
```

这样一来，我们既约束了这个变量的类型，又没有丢失其声明时自动携带的类型信息，很难说不完美。

> 你可能会觉得 satisfies 这个名字不太好理解，其实当时还有另一个候选的名字 implements，但由于它实际上已经是一个 JavaScript（ECMAScript）关键字，同时也已经被使用在 TypeScript Class 对抽象类以及接口的实现声明中，为了避免造成不必要的困惑，最后还是 satisfies 成功胜出。

你以为我们讲完了这个新特性了？当然没有，下面是夹带私货时间。

如果你对 TypeScript 中的类型层级有一定了解，或是使用过其他静态类型的编程语言，很容易发现这里其实还蕴藏着一个类型层级的上下级关系：推导得到的类型实际上是我们标注类型的子类型：

```typescript
type A = Record<Colors, string | RGB>;

type B = {
  red: RGB;
  green: string;
  blue: RGB;
};

declare let a: A;
declare let b: B;

a = b; // √，说明子类型关系成立
b = a; // ×
```

这个其实很好理解，类型 B 的键名字面量可以被视为 Colors 的子类型（联合类型本身也是自己的子类型），而键值类型也满足约束。

那么当我们写出 `satisfies Record<Colors, string | RGB>` 时，实际上是在进行类型的向上转换，即 upcast。

说到这里有的同学肯定会想到，我们也可以使用类型断言来实现类型的向上转换啊，即 `as Record<Colors, string | RGB>`。而如果你试了，就会明白为啥我不提它了。

首先，使用类型断言的本质还是让显式提供的类型信息完全覆盖推导得到的类型信息，从这一点来看就可以毙掉了。而更可怕的一点是，类型断言是允许你把不正确的值断言成提供的类型信息的：

```typescript
const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
} as Record<Colors, string | RGB>;

palette.blue.includes();
```

我们并没有正确地实现这个类型，却为后续的代码调用提供了这么一个错误的类型信息，相当于埋下了一颗定时炸弹，因此使用类型断言来实现 upcast 其实是一个不怎么安全的操作。

就像 TypeScript 发现过于自由的 any 类型可能会带来问题一样，更安全的 Top Type —— unknown 类型加入解决了使用顶级类型时的安全隐患。现在，为了让 upcast 操作也能更安全一些，我们又迎来了 satisfies ，可以确定在不远的未来，它必定会成为你最好的类型工具之一。

你以为又完了？还没呢。就像 Top Type 对应到 Bottom Type 一样，upcast 其实也可以对应到 downcast ，即向下断言，但不同的是，upcast 很多时候是自动实现的，而 downcast 则必须要手动实现，以及附加类型检查。

```typescript
class Animal {
  eat() {}
}

class Dog extends Animal {
  bark() {}
}

const dog = new Dog();
dog.eat();
```

我们调用的 eat 方法，实际上来自于 Animal 类型，这个时候其实我们就是隐式地执行了 upcast ，将其类型向上转换到了 Animal 类型，也就是将猫狗都视为动物。这实际上也符合抽象原则和 SOLID 原则中的里氏替换原则——在某些时候，我并不需要知道这个动物到底是什么物种，只需要知道它是动物而不是植物就行了。同时，只要你这个地方需要的是 Animal 类型，那我只要给的是其子类型，不管是什么动物，都能确保其工作正常。

但如果我们想要执行 downcast，比如从 Animal 类型向下转换到其子类型，这个时候就可能出现问题，因为我们无法确定此时它是否真的是对应的子类型，所以通常需要配合类型守卫：

```typescript
class Cat extends Animal {
  meow() {}
}

const wtfIsThisAnimal = new Animal();

if (wtfIsThisAnimal instanceof Dog) {
  wtfIsThisAnimal.bark();
}

if (wtfIsThisAnimal instanceof Cat) {
  wtfIsThisAnimal.meow();
}
```

这两个概念在 Java 中也是类似的，upcast 会自动执行，而 downcast 则需要通过检查，否则就会抛出错误（`ClassCastException`）：

```java
Animal animal1 = new Animal();
Animal animal2 = new Dog();

Dog notADog = (Dog) animal1;
Dog actuallyADog = (Dog) animal2;
```

#### 包含 .ts 后缀的导入路径支持

> 此特性并没有包含在 4.9 beta 中，而是被作为一个 4.9 整体的工作项，这里属于提前介绍。

TypeScript 中使用 moduleResolution 配置项来进行导入模块的路径解析，其常用的配置值 `'node'` 会保持与 NodeJs 相同的解析策略，如对相对路径 `./foo` 的解析，会首先尝试解析 `<root>/<project>/src/foo.ts` 与 `<root>/<project>/src/foo.d.ts`，也即是说导入路径无需携带后缀名 `.ts`。

而在 4.9 版本中，通过 `--noImplicitSuffix` 选项（具体的选项配置还未最终确定）来修改了这一行为，在启用此选项时，导入路径必须显式携带 `.ts` 后缀才能正常解析，而不是依赖 moduleResolution 。

这一行为同样有助于对齐 deno 中的导入方式，如：

```typescript
import { writableStreamFromWriter } from 'https://deno.land/std@0.156.0/streams/mod.ts';
```

#### 单文件级别配置

> 此特性并没有包含在 4.9 beta 中，而是被作为一个 4.9 整体的工作项，这里属于提前介绍。

TypeScript 支持通过 `@ts-nocheck` 与 `@ts-check` 指令来更改单个文件内的检查策略，如在关闭 `checkJs` 的情况下使用 `@ts-check` 来启用对少数几个 JS 文件的检查，或者在开启的情况下使用 `@ts-nocheck` 禁用对某些 JS 文件的检查。但这两个指令仅仅能影响是否检查，而无法影响检查的具体配置。

因此，4.9 beta 版本引入了单文件级别的 tsconfig 配置，使用 `@ts-config value` 的形式，如以下示例：

```typescript
// @ts-strict
// @ts-noUnusedLocals
// @ts-strictNullChecks
// @ts-noPropertyAccessFromIndexSignature false
```

你可以查看 [#49886](https://github.com/microsoft/TypeScript/pull/49886) 来了解所有已被支持的单文件配置。类似于 `@ts-check`，这些指令必须被放在文件的顶部才能生效。

### 对未列出属性的类型收窄增强

在面对联合类型时，我们经常使用类型守卫的方式来显式地提供类型信息，来帮助修正对应分支的类型控制流分析上下文。而在类型守卫中，最常使用的则是 `in` 关键字：

```typescript
interface LoginUser {
  userId: string;
  invitor: string;
}

interface Visitor {
  visitorId: string;
  from: string;
}

function checkUser(user: LoginUser | Visitor): string {
  if ('userId' in user) {
    return user.invitor;
  } else {
    return user.from;
  }
}
```

然而 in 关键字在某些时候也存在着能力的不足，如以下这个示例：

```typescript
interface LoginUser {
  userId: string;
  invitor: string;
}

interface Visitor {
  visitorId: string;
  from: string;
}

function checkUser(info: { user: unknown }): string {
  const user = info.user;
  if (user && typeof user === 'object') {
    // 类型 object 上不存在属性 userId
    if ('userId' in user && typeof user.userId === 'string') {
      // 类型 object 上不存在属性 invitor
      return user.invitor;
    }
  }
}
```

正常来说，user 已经被收窄到 object 类型，同时也满足了 `userId in user` 这个条件，为什么在 `typeof user.userId === 'string'` 还会有属性不存在的报错？

这是因为 in 操作符只会严格缩小到当前需要的检查类型，而 userId 并没有在 user 类型上列出（unlisted），所以 user 仍然只会是 object 类型。而在 4.9 版本，现在面对这种情况， in 操作符会将检查类型缩小到 `object & Record<"userId", unknown>` 类型，这样就能够支持未列出属性的类型守卫了。

另外，4.9 版本现在也会约束 in 操作符的左侧必须是 string / number / symbol 类型，以及右侧必须是 object 类型。

### 对 NaN 类型的相等检查

在 JavaScript 中 NaN 是一个特殊的数值类型值，ECMA262 标准中明确规定了它是一个基于 _IEEE 754-2019_ 规范（_IEEE Standard for Floating-Point Arithmetic_. Institute of Electrical and Electronic Engineers, New York (2019)）的“非正常数字”值。也就是说，它还是数值类型的值，但不是正常数字，或者说非数字。是不是有点混乱？在 ECMAScript 标准中规定了 Number 类型为上面提到的 IEEE 754 中 64 位浮点数实现，而 NaN 也就是遵循此标准的一个数字，并同样有重要的作用，如作为 `0/0` 这类运算的结果。

> 另外，null 类型是一个 Null 类型的值（也是唯一一个），虽然 typeof null 的结果是 object ，但这更多是历史包袱的原因。

而 NaN 实际上与任何类型的任何值都不等价（包括 NaN 本身），因此在判断一个值是不是 NaN 时，我们需要使用 `Number.isNaN` 而不是 `value === NaN` 。

TypeScript 4.9 版本新增了错误使用等价判断方式的提示：

```typescript
// 此表达式将始终返回 false，你是否指 Number.isNaN(value) ?
if (value === NaN) {
}
```

TypeScript 这几个版本一直在升级这一类贴心的小功能，如 4.8 版本对象字面量值与数组字面量值的全等比较提示：

```typescript
const obj = {};

// 此语句始终将返回 false，因为 JavaScript 中使用引用地址比较对象，而非实际值
if (obj === {}) {
}
```

### 其他更新

#### Promise.resolve 的类型更新

TypeScript 在 4.5 版本引入了新的工具类型 Awaited ，用于提取一个 Promise resolve 之后的类型：

```typescript
type Awaited<T> = T extends null | undefined
  ? T
  : T extends object & {
      then(onfulfilled: infer F): any;
    }
  ? F extends (value: infer V, ...args: any) => any
    ? Awaited<V>
    : never
  : T;
```

在 4.5 版本引入此类型时，TS 已经替换了一批相关方法的类型签名，如 `Promise.all` 等方法，而在 4.9 版本中，则对 `Promise.resolve` 的类型签名也进行了替换：

```typescript
interface PromiseConstructor {
  resolve<T>(value: T | PromiseLike<T>): Promise<T>;
}

// 更新为
interface PromiseConstructor {
  resolve<T>(value: T): Promise<Awaited<T>>;
  resolve<T>(value: T | PromiseLike<T>): Promise<Awaited<T>>;
}
```

这样改造后， `Promise.resolve` 将尽可能返回一个 resolve 后的 Promise 类型值。

#### 完全保留 JavaScript 文件中的导入

TypeScript 在编译过程中会进行类型检查与语法降级的工作，而在类型检查中，假设当前正在处理的已经是 JavaScript 文件，此时一旦 TS 检测到了一个仅作为类型使用的导入，就会将这个导入从 JavaScript 文件中移除：

```javascript
import { someValue, SomeClass } from 'some-module';

/** @type {SomeClass} */
let val = someValue;
```

这里 SomeClass 仅被作为 JSDoc 的类型标注使用，因此是可以直接从导入语句中被移除的。但这么做可能导致的问题是如果 JSDoc 类型标注不完全准确，就会导致这一擦除行为也表现异常。

因此，现在 JavaScript 文件中的导入语句将会被完全保留。

全文完，我们 5.0 beta 版本见:-)。
