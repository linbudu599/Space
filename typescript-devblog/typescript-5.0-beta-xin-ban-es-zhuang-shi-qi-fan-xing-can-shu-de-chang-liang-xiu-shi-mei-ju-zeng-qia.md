---
description: '2023-01-26'
---

# TypeScript 5.0 beta：新版 ES 装饰器、泛型参数的常量修饰、枚举增强等

TypeScript 已于 2023.01.26 发布 5.0 beta 版本，你可以在 [5.0 Iteration Plan](https://github.com/microsoft/TypeScript/issues/51362) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash

$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [**JavaScript and TypeScript Nightly**](http://link.zhihu.com/?target=https%3A//marketplace.visualstudio.com/items%3FitemName%3Dms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。

本篇是笔者的第六篇 TypeScript 更新日志，上一篇是 「TypeScript 4.9 beta 发布：鸽置的 ES 装饰器、satisfies 操作符、类型收窄增强、单文件级别配置等」，你可以在此账号的创作中找到（或在掘金/知乎搜索**林不渡**），接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

**补充说明：TypeScript 并不使用 Semantic Version 规范，这意味着你不能将 x.0 作为一个 Major 版本——因为它可能并不包含破坏性更新，你也不能将 5.x 作为一个 Minor 版本——因为它可能就包含破坏性更新。**

### ECMASCript 装饰器

ES 新版装饰器终于在 5.0 中成功 Landing（部分关于其后续迭代的讨论，请参考 [#50820](https://github.com/microsoft/TypeScript/pull/50820)），以下对新版装饰器 API 的介绍来自于笔者此前发表的 [**ECMAScript 双月报告：装饰器提案进入 Stage 3**](https://link.zhihu.com/?target=https%3A//link.juejin.cn/%3Ftarget%3Dhttps%3A%2F%2Fmp.weixin.qq.com%2Fs%2F6PTcjJQTED3WpJH8ToXInw) 。

一个最基本的装饰器类型定义大致是这样的：

```typescript
type Decorator = (
  value: Input,
  context: {
    kind: string;
    name: string | symbol;
    access: {
      get?(): unknown;
      set?(value: unknown): void;
    };
    isPrivate?: boolean;
    isStatic?: boolean;
    addInitializer?(initializer: () => void): void;
  }
) => Output | void;
```

value 为这个装饰器应用处的类或类成员的值，而 context 则包含了这一被装饰的值的上下文信息。这两个参数都基于装饰器实际应用的位置来决定，如果装饰器的调用返回了一个值（Output），那么被装饰位置的值会被这个返回值替换掉。

对于 context 参数，我们先对其内部的属性做一个简单介绍：

* kind，被装饰的值的类型，如 'class' / 'method' / 'field' 等，这一属性可以被用来校验装饰器被应用在了正确的位置，或者在同一个装饰器中，基于实际应用位置执行不同的装饰逻辑。
* name，被装饰的值的名称，如类名、属性名、方法名等。
* access，其包含了这个值的 getter 与 setter，我们会在下面详细介绍。
* isStatic 与 isPrivate，在装饰器应用于类成员时提供这一成员的访问性修饰符信息。
* addInitializer，可以通过这个属性添加要在类实例化时执行的逻辑。

需要注意的是，除了语义与参数的变化，新版的装饰器在调用语法上也进行了一些调整：

* 类表达式现在也可以应用装饰器了，如：

```typescript
const Foo = @deco class {
  constructor() {}
}
```

* 装饰器与 export 关键字一同应用的方式调整为：

```typescript
export default @deco class Foo { }
```

以下基于当前的装饰器类型，对装饰器的基本能力进行简要介绍，类型定义来自于 5.0.0 beta 版本中的 `lib.decorator.d.ts` 声明文件，为了可读性进行了略微修改。另外，`context.access` 属性目前暂时被禁用，正在等待 [#494](https://github.com/tc39/proposal-decorators/issues/494) 的讨论得出结果。

#### 类装饰器

类装饰器的类型定义如下：

```typescript
type ClassDecorator = (
  value: Function,
  context: {
    kind: 'class';
    name: string | undefined;
    addInitializer(initializer: (this: Class) => void): void;
  }
) => Function | void;
```

value 为被装饰的 Class，你可以通过返回一个新的 Class 来完全替换掉原来的 Class。或者由于你能拿到原先的 Class，你也可以直接返回一个它的子类：

```typescript
function logged(value, { kind, name }) {
  if (kind === 'class') {
    return class extends value {
      constructor(...args) {
        super(...args);
      }
    };
  }
}

@logged
class C {}
```

#### 类方法装饰器

类方法装饰器的类型定义如下：

```typescript
type ClassMethodDecorator = (
  value: Function,
  context: {
    kind: 'method';
    name: string | symbol;
    static: boolean;
    private: boolean;
    addInitializer(initializer: (this: This) => void): void;
  }
) => Function | void;
```

其 value 参数为被装饰的类方法，可以通过返回一个新的方法来直接在原型层面代替掉原来的方法（对于静态方法则在 Class 的层面替换）。或者你也可以包裹这个原来的方法，执行一些额外的逻辑：

```typescript
function logged(value, { kind, name }) {
  if (kind === 'method') {
    return function (...args) {
      const ret = value.call(this, ...args);
      return ret;
    };
  }
}

class C {
  @logged
  m(arg) {}
}
```

#### 类属性装饰器

类属性装饰器的类型定义如下：

```typescript
type ClassFieldDecorator = (
  value: undefined,
  context: {
    kind: 'field';
    name: string | symbol;
    access: { get(): unknown; set(value: unknown): void };
    isStatic: boolean;
    isPrivate: boolean;
  }
) => (initialValue: unknown) => unknown | void;
```

不同于上面的几种装饰器，属性装饰器的 value 并不是被装饰的属性的值，而是一个 undefined。如果要获取被装饰的属性值，你可以让属性装饰器返回一个函数，这个函数会在属性被赋值时调用，拿到初始值作为入参，并可以返回一个新的值作为实际的赋值。

```typescript
function logged(value, { kind, name }) {
  if (kind === 'field') {
    return function (initialValue) {
      return 599;
    };
  }

  // ...
}

class C {
  @logged x = 1;
}

new C().x; // 599
```

属性访问器装饰器与 Auto Accessor 的介绍请参考上面的 TC39 会议报告，这里不再展开。同时，目前新版装饰器中并不存在参数装饰器。

如果你想了解更多新版装饰器的实际使用，可以参考 [with-new-decorators](https://github.com/linbudu599/with-new-decorators)，其中包括了使用 Babel 和 TypeScript 5.0 中的新版装饰器应用。

另外，你也可以参考 [Mustard](https://github.com/LinbuduLab/Mustard)，这是一个基于新版装饰器，不依赖元数据的命令行应用构建库（Command Line App Builder），它的使用大概是这样的：

```typescript
import { MustardFactory, MustardUtils } from 'mustard-cli';
import { RootCommand, Option, App, Utils, Input } from 'mustard-cli/decorator';
import { CommandStruct, MustardApp } from 'mustard-cli/cli';

type Template = 'typescript' | 'javascript';

@RootCommand()
class RootCommandHandle implements CommandStruct {
  @Option('dry', 'd', 'dry run command to see what will happen')
  public dry = false;

  @Option('template', 't', 'template to use')
  public template: Template = 'typescript';

  @Input('directory to create the project in')
  public dir: string = './';

  @Utils()
  public utils!: MustardUtils;

  public run(): void {
    console.log(this.utils.colors.green('Hello World'));
  }
}

@App({
  name: 'awesome-mustard-app',
  commands: [RootCommandHandle],
})
class Project implements MustardApp {
  onError(error: Error): void {
    console.log(error);
  }
}

MustardFactory.init(Project).start();
```

以上的应用构建了一个使用 `awesome-mustard-app` 作为命令的 CLI 应用，它的调用方式是这样的：

```bash
$ create-awesome-app my-awesome-app --dry --template=typescript
```

如果你想体验一下 Mustard CLI，可以执行以下命令：

```bash
$ pnpx create-mustard-app

$ cd mustard-app
$ npm install
$ npm run dev
```

目前 Mustard 仍在进一步完善中，[文档](https://mustard-cli.netlify.app/) 也仍在编写中，欢迎随手点个 star \~

另外一点关于新版装饰器需要说明的是，旧版装饰器的好伙伴反射元数据（Reflect Metadata）目前并不支持与新版装饰器一同使用，笔者的猜想是反射元数据提案可能会在 3 月或更晚的 TC39 会议上进行讨论，大概率也需要进行数轮修改才能推进到 Stage 3。因此，如果你想提前开始使用新版装饰器，短期内是指望不了元数据能力的。

然而我们知道，元数据是基于装饰器实现依赖注入的重要手段，基于旧版装饰器的框架基本都是一个套路：类的成员注入元数据，然后由工厂方法在实例化这个类的同时按照元数据来初始化成员。比如使用装饰器的 NodeJs 框架中会这么写：

```typescript
@Controller('/user')
class UserController {
  @Inject()
  userService: UserService;

  @Get('/query')
  async queryUser() {}

  @Post('/create')
  async addUser() {}
}
```

`UserService` 会作为类型的元数据被注入到 `userService` 上，并在实例化时注入一个 UserService 实例。同时 queryUser 和 addUser 会分别被注册为 `GET /query` 和 `POST /create` 的请求处理方法。缺少了元数据的类型信息注入，在新版装饰器中我们暂时无法优雅地实现 `UserService` 到 `userService` 的注入。

而在 Mustard 中，我们的应用场景暂时不依赖类型信息来实现实例属性注入，而是只需要做命令行参数到属性名的映射，然后将值注入即可。我们使用了 `context.addInitializer`，将装饰器收集到的属性名、初始值等信息强行替换掉实例内的属性值，再由工厂方法按照这个 Initializer 描述来真正进行实例的初始化。

最后，旧版的 `--experimentalDecorators` 选项将会仍然保留，如果启用此配置，则仍然会将装饰器视为旧版，新版的装饰器无需任何配置就能够默认启用。

### 泛型参数的常量修饰

在此前，函数中的泛型参数推导只能推导到基础类型一级（即比字面量类型高出一个层级的类型），如 `string` 、`string[]` 这样：

```typescript
declare function foo<T>(x: T): T;

foo('linbudu'); // string
foo([1, 2, 3]); // <number[]
// { name: string; techs: string[]; }
foo({ name: 'linbudu', techs: ['nodejs', 'typescript', 'graphql'] });
```

这其实类似于使用 `let` 声明变量时的自动类型推导表现。

TypeScript 5.0 新增了对泛型参数的常量修饰（基本等价于常量断言），被修饰的泛型参数在进行类型信息推导时，将推导到尽可能精确的字面量类型层级：

```typescript
declare function foo<const T>(x: T): T;

foo('linbudu'); // 'linbudu'
foo([1, 2, 3]); // readonly [1, 2, 3]
// { name: 'linbudu'; techs: readonly ['nodejs', 'typescript', 'graphql'] }
foo({ name: 'linbudu', techs: ['nodejs', 'typescript', 'graphql'] });

// 如果不使用常量修饰，等价于此效果
foo({ name: 'linbudu', techs: ['nodejs', 'typescript', 'graphql'] } as const);
```

可以看到这里对于数组类型的类型推断，其实基本等价于常量断言后的类型：每一级的数组类型都加上了 readonly 修饰。

当被常量修饰的泛型参数为数组类型时，如果其泛型约束不包含 readonly，则推导出的类型将回归到泛型约束来维持其可变状态，否则才会是预期的常量推导：

```typescript
declare function foo<const T extends readonly string[]>(args: T): T;

declare function bar<const T extends string[]>(args: T): T;

// [readonly ["a", "b", "c"]
foo(["a", "b" ,"c"]);
// string[]
bar(["a", "b" ,"c"]);
```

### 枚举增强

TypeScript 5.0 中对枚举进行了一次全面的能力增强，移除了此前诸如「枚举计算成员必须位于字面量成员之后」、「仅允许在数字枚举中定义计算成员」、「常量枚举中不允许包含变量或表达式」的限制：

```typescript
const BaseValue = 10;
const Prefix = '/data';

enum Values {
  First = BaseValue,
  // 此前会提示「枚举成员必须具有初始化表达式」，现在会正确从 10 开始累加
  Second,
  Third,
}
enum Routes {
  // 此前会提示「只有数字枚举可具有计算成员，但此表达式的类型为“string”」
  Parts = `${Prefix}/parts`,
  Invoices = `${Prefix}/invoices`,
}

const enum ConstValues {
  // 此前会提示「常量枚举成员初始值设定项只能包含字面量值和其他计算的枚举值」
  First = BaseValue, // 10
  Second, // 11
  Third, // 12
}
const enum ConstRoutes {
  // 此前会提示「常量枚举成员初始值设定项只能包含字面量值和其他计算的枚举值」
  Parts = `${Prefix}/parts`, // "/data/parts"
  Invoices = `${Prefix}/invoices`, // "/data/invoices"
}
```

这一变化的主要原因在于，此前 TypeScript 中存在数字枚举和字符串枚举两种类型的枚举，其中数字枚举仅允许数字类型与计算属性成员，不允许字符串类型，而字符串枚举又仅允许数字或字符串枚举成员：

```typescript
const BaseValue = 10;
const Prefix = '/data';

enum Values {
  First = BaseValue,
  // 枚举成员必须具有初始化表达式。
  Second,
  Third,
}
enum Routes {
  // 只有数字枚举可具有计算成员，但此表达式的类型为“string”。
  Parts = `${Prefix}/parts`,
  Invoices = `${Prefix}/invoices`,
}
```

而在 5.0 版本中将这两种枚举合并成了单一的、功能更强大的枚举类型，其成员允许任何常量或表达式计算，并仅为常量成员赋予字符串。同时，现在一个枚举类型将被视为其所有成员类型组成的联合类型，如伪代码 `type Enum = Enum.A | Enum.B | Enum.C`。

### --verbatimModuleSyntax 配置

我们知道，TS 文件到 JS 文件的编译过程主要包括三件事：**类型信息擦除**、**语法降级**、**声明文件生成**。关于类型信息擦除，你应该能立刻想到包括类型定义和类型签名，但实际上这里还包含着常常被忽略的类型导入。

```typescript
import { User } from './user.model';

const user: User = {};
```

在这个例子中，我们很容易确定 User 只被作为类型使用，需要被移除（否则如果运行时不存在一个 User Class，就会出错了），那么，如果 `user.model.ts` 中有这么一行代码：

```typescript
// 额外的副作用
setupUserModelMiddleware();

export class User {}
```

这个时候，如果把导入语句移除，可能就会导致代码运行时出现异常。

一直以来，为了避免对导入语句的移除影响运行时代码，TypeScript 使用了多种方式来确定一条导入语句能否被移除，如检查对导入的使用，以及其在导出文件中是如何声明的。

为了简化类型导入的判断工作，TypeScript 此前引入了 `import type` 语法来声明仅类型导入，或成员级别的仅类型导入：

```typescript
import type * as UserTypes from "./user";

import { type UserModel, UserMiddleware } from "./user";
```

对应的，还有一系列配置来进一步细粒度地控制行为，如 `--importsNotUsedAsValues` 用于确认类型导入的使用（仅类型导入需要被显式标记，而未被使用的值导入仍然将会保留），`--preserveValueImports` 用于显式避免部分导入语句的移除（所有值导入都将被完整保留，避免 TypeScript 无法检测其使用方式的情况）等等。

而在 5.0 版本，TypeScript 引入了新的配置 `--verbatimModuleSyntax` 来进一步简化这些情况，它的作用就简单多了：**所有非仅类型导入/导出都会被保留，而仅类型导入/导出都会被移除。**

### moduleResolution 相关

> 这一部分包括了 `--moduleResolution bundler` 与 `Resolution Customization Flags` 的相关介绍。

TypeScript 自 [4.5 版本](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5-beta/) 来一直在持续改进 NodeJs 中的 ESM 支持，先后在 4.5 beta 与 4.7 正式中引入了 `.mts`、`.cts` 扩展名，支持了 `package.json` 中的 exports, imports, type 等字段，以及新的 `compilerOption.module` 与 `compilerOptions.moduleResolution` 值：`node16` 与 `nodenext`，这些能力很好地改进了 TS + NodeJs + Pure ESM 的研发体验，但实际上，前端 er 们并不会仅仅使用 tsc 来进行编译，而其他的编译工具其实并没有这么多弯弯绕绕。

举例来说，NodeJs 中的 ESM 强制要求你的相对导入路径携带扩展名，即 `import ns from "./mod.mjs"`（你也可以使用 `--experimental-specifier-resolution=node` 配置来启用自动地路径解析），这主要是为了贴合 NodeJs 在服务器环境下的性能表现。然而大部分的构建工具其实不要求你这么做，它会融合 ESM 与 CJS 的模块解析策略。

在 5.0 beta 版本，TS 引入了新的 `moduleResolutio: bundler` 配置，它会使用 NodeJs 的模块解析策略，支持 ESM 语法，但不会强制你使用 ESM 的这些严格解析规则。同时 5.0 还引入了一系列细粒度的配置项来便于各个运行时与构建工具按照自己的需求进行调整：

* `--allowImportingTsExtensions`，此配置启用后，在相对导入 TS 文件时就能够携带上扩展名（`.ts`, `.mts`, `.tsx`，不包括 `.cts` ，因为 ESM 才是一家人）。但需要注意的是需要同时启用 `--noEmit` 或者 `--emitDeclarationOnly`，这是因为这些文件导入路径还需要被构建工具进行处理后才能正常使用，运行时本身是无法
* `--resolvePackageJsonExports`，`--resolvePackageJsonImports`，这两个配置将分别强制 TS 在读取 来自 `node_modules` 中的导入时去解析 `package.json` 中的 exports 与 imports 字段。在 `moduleResolution` 被指定为 node16 / nodenext / bundler 时默认启用。
*   `--allowArbitraryExtensions`，启用此配置后，TS 在导入一个非 JS/JSX/TS/TSX 扩展名的文件时，也会自动去查找其类型声明。如导入 `style.css` 时将尝试加载 `style.d.css.ts` 声明文件：

    ```css
    /* style.css */
    .app-container {
    }

    .app-main-title {
    }
    ```

    ```typescript
    // style.d.css.ts
    declare const css: {
      appContainer: string;
      appMainTitle: string;
    };

    export default css;
    ```

    你可能会想，为什么是 `.d.css.ts`，而不是 `.css.d.ts` ？ 这是因为 `file.d.ts` 通常被视为 `file.js/jsx` 的声明文件，也就是 `.css` 其实被视为了不完整的 JS 文件名，这实际上是错误的行为。

    这一配置主要是为了避免在支持这些导入的运行时或者构建工具中产生类型报错（此前我们通常通过 `declare module '*.css'` 来实现），它对业务开发确实有着明显的意义，社区可能又要为此涌现出一批新活了。
*   `--customConditions`，NodeJs 支持在 `package.json` 的 exports 中指定 `import` / `require` / `node` / `default` 等值来设定其在不同条件(环境)下的文件入口。而这一配置则是为了更灵活地指定条件，如你可以在 `tsconfig.json` 中这样配置：

    ```json
    {
      "compilerOptions": {
        "target": "es2022",
        "moduleResolution": "bundler",
        "customConditions": ["only-for-linbudu"]
      }
    }
    ```

    这样若是你的 npm 包 exports 中指定了这一条件，则它会将其视为最高优先级：

    ```json
    {
      "name": "@linbudu/pkg",
      "exports": {
        ".": {
          "only-for-linbudu": "./private.mjs",
          "node": "./public.mjs",
          "import": "./public.mjs"
        }
      }
    }
    ```

### 其他

#### 类型全量导出

现在，你可以使用 `export type * from 'module'` 或者 `export type * as namespace from 'module'` 来导出类型了：

```typescript
// models/user.model.ts
export class UserModel {}

// models/index.ts
export type * as models from './user.model.ts';

// app.ts
import { models } from './models';

// √ 仅作为类型使用
const userModel: UserModel = {};

// × 不能被作为值使用
const userModel = new models.UserModel();
```

#### JSDoc 中的 @satisfies 与 @overload

TypeScript 4.9 版本中引入了用于进行安全 upcast 操作的 satisfies 操作符（参考 [TypeScript 4.9 beta 发布：鸽置的 ES 装饰器、satisfies 操作符、类型收窄增强、单文件级别配置等](https://zhuanlan.zhihu.com/p/568294633)），而在 5.0 版本中，为了支持在 JavaScript 文件中使用 JSDoc 进行类型检查的使用方式，现在你可以使用 `@satisfies` 标签来检查类型，但同时在上下文中保持使用从原先值推导出的类型（这也是 satisfies 和 类型断言最大的差异）：

```javascript
// @ts-check

// 类型“{ name: string; }”不满足预期类型“{ name: string; version: string; }”。
/**
 * @satisfies {{name:string, version:string}}
 */
let pkg = {
  name: 'mustard-cli',
};
```

另外一个在本次被添加的 JSDoc 标签是 `@overload`，它用于显式标明每一个函数的重载签名，从而配合类型检查，如以下的例子：

```javascript
// @ts-check

/**
 * @param {string} value
 * @return {void}
 */

/**
 * @param {number} value
 * @param {number} [maximumFractionDigits]
 * @return {void}
 */

/**
 * @param {string | number} value
 * @param {number} [maximumFractionDigits]
 */
function printValue(value, maximumFractionDigits) {}
```

`printValue("hello!", 123)` 是一个错误的重载调用，但却没有提示报错信息，但如果为重载签名添加 `@overload` 标签，TS 就能够依次检查其是否有符合的重载调用：

```typescript
// @ts-check

/**
 * @overload
 * @param {string} value
 * @return {void}
 */

/**
 * @overload
 * @param {number} value
 * @param {number} [maximumFractionDigits]
 * @return {void}
 */

/**
 * @param {string | number} value
 * @param {number} [maximumFractionDigits]
 */
function printValue(value, maximumFractionDigits) {}

// 类型“string”的参数不能赋给类型“number”的参数。
printValue('hello!', 123); // error!
```

#### 废弃功能

为了更好地迎接未来的 ECMAScript 演进，TypeScript 5.0 引入了对语言能力或配置项的废弃计划，以 `keyofStringsOnly` 配置为例，在 5.0 版本开始，启用此配置将会获得一条警告：

```bash
TS9998: Flag 'keyofStringsOnly' is deprecated and will stop functioning in TypeScript 5.5. Specify 'ignoreDeprecations: "5.0"' to silence this error
```

在下一阶段（如 5.5 版本），你仍然可以指定 `keyofStringsOnly` ，它也不会抛出任何错误，但实际上它已经不再有作用。在最后一个阶段（如 6.0 版本），指定此配置将会抛出一个错误。

在第一阶段，你可以通过设置 `ignoreDeprecations: "5.0"` 来关闭所有由于使用在 5.0 版本开始废弃的功能而产生的警告。

已经确定在 5.0 版本将开始逐步废弃的配置项包括 `charset`，`noImplicitUseStrict`，`keyofStringsOnly`，`noFallthroughCasesInSwitch` 等，以及 `target: ES3` 与 `module: umd/system/amd`。

#### tsconfig.json 多继承

TypeScript 5.0 支持了 tsconfig.json 的 extends 配置的数组类型，用于同时继承一组已有的规则，这一能力使得你能够将自己的共享配置进一步拆解，再依据实际情况进行组合：

```json
{
  "extends": [
    "./tsconfig.modern.json",
    "./tsconfig.node.json",
    "./tsconfig.strict.json"
  ]
}
```

以上这个配置集成了包含现代语言特性配置、包含 Node 应用配置以及包含严格检查的配置文件。

#### 单文件级别配置

TypeScript 5.0 现在支持单文件级别的配置，如你只想为当前文件启用严格检查，其他文件仍然使用全局配置，可以这么做：

```typescript
// @ts-strict

// 报错：参数 input 隐式具有 any 类型
const handler = (input) => input + 1;
```

目前支持的规则：

* `strict`
* `noImplicitAny`
* `strictNullChecks`
* `strictFunctionTypes`
* `strictBindCallApply`
* `strictPropertyInitialization`
* `noImplicitThis`
* `useUnknownInCatchVariables`
* `alwaysStrict`
* `noUnusedLocals`
* `noUnusedParameters`
* `exactOptionalPropertyTypes`
* `noImplicitReturns`
* `noFallthroughCasesInSwitch`
* `noUncheckedIndexedAccess`
* `noImplicitOverride`
* `noPropertyAccessFromIndexSignature`

其配置项也均为小驼峰转中划线，如 `@ts-no-property-access-from-index-signature`。

全文完，我们 TS 5.1 见:)

