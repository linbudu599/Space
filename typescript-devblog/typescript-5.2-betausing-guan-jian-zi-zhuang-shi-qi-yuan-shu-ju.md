---
description: '2023-07-01'
---

# TypeScript 5.2 beta：using 关键字、装饰器元数据

TypeScript 已于 2023.0630 发布 5.2 beta 版本，你可以在 [5.2 Iteration Plan](https://github.com/microsoft/TypeScript/issues/54298) 查看所有被包含的 Issue 与 PR。如果想要抢先体验新特性，执行：

```bash
$ npm install typescript@beta
```

来安装 beta 版本的 TypeScript，或在 VS Code 中安装 [JavaScript and TypeScript Nightly](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-typescript-next) 来更新内置的 TypeScript 支持。本篇是笔者的第八篇 TypeScript 更新日志，上一篇是 「TypeScript 5.1 beta 发布：函数返回值类型优化、Getter/Setter类型优化、JSX 增强」，你可以在此账号的创作中找到（或在掘金/知乎搜索林不渡），接下来笔者也将持续更新 TypeScript 的 DevBlog 相关，感谢你的阅读。

> 另外，由于 beta 版本与正式版本通常不会有明显的差异，这一系列通常只会介绍 beta 版本而非正式版本。

### using 关键字：显式资源管理

TypeScript 团队成员 [rbuckton](https://github.com/rbuckton) 主导的 TC39 提案 [proposal-explicit-resource-management](https://github.com/tc39/proposal-explicit-resource-management) 在 2023 年 3 月的 TC39 会议上已经进入 Stage 3 阶段，这意味着它大概率将成为 ECMAScript Next 的一员，所以 rbuckton 在 TypeScript 5.2 beta 中也顺理成章地引入了相关支持。Explict Resource Management 提案旨在为 JavaScript 中引入显式的资源管理能力，如对内存、I/O、文件描述符等资源的分配与显式释放。实际上，所谓的“显式”对应的是此前如 WeakSet 与 WeakMap 这样，会由 JS 运行时作为垃圾回收进行的“隐式”工作。显式资源管理意味着，用户可以主动声明块级作用域内依赖的资源，通过 [Symbol.disposable](https://github.com/zloirock/core-js/blob/901efc66ecfd612f107976ec6a2cd6f426b70975/packages/core-js/modules/esnext.symbol.dispose.js) 这样的命令式或 using 这样的声明式，然后在离开作用域时就能够由运行时自动地释放这些标记的资源。在此前，JavaScript 中并不存在标准的资源管理能力，如以下的 Generator 函数：

```typescript
function * g() {
  const handle = acquireFileHandle();
  try {
    ...
  }
  finally {
    handle.release(); // 释放资源
  }
}

const obj = g();
try {
  const r = obj.next();
  ...
}
finally {
  obj.return(); // 显式调用函数 g 中的 finally 代码块
}
```

在我们执行完毕生成器函数后，需要使用 obj.return() 来显式地调用 g 函数中的 finally 代码块，才能确保文件句柄被正确释放。类似的，还有 NodeJs 中对文件描述符的操作（fs.openSync）：

```typescript
export function doSomeWork() {
    const path = ".some_temp_file";
    const file = fs.openSync(path, "w+");

    try {
        if (someCondition()) {
            return;
        }
    }
    finally {
        fs.closeSync(file);
        fs.unlinkSync(path);
    }
}
```

这些资源释放逻辑其实都是不必要的，正常来说，这些向操作系统申请的资源应该在代码离开当前作用域时就被释放掉，而不需要显式操作。因此此提案提出使用 using 关键字，来声明仅在当前作用域内使用的资源：

```typescript
function * g() {
  using handle = acquireFileHandle(); // 定义与代码块关联的资源
}

{
  using obj = g(); // 显式声明资源
  const r = obj.next();
} // 自动调用释放逻辑


{
  using handlerSync = openSync();
  await using handlerSync = openAsync(); // 同样支持异步的资源声明
} // 自动调用释放逻辑
```

需要注意的是，using 关键字并不是语言底层的新功能，它更像是语法糖，比如我们看 Babel 插件的编译结果：

```javascript
// 输入
using handlerSync = openSync();
await using handlerAsync = await openAsync();

// 输出
try {
  var _stack = [];
  var handlerSync = babelHelpers.using(_stack, openSync());
  var handlerAsync = babelHelpers.using(_stack, await openAsync(), true);
} catch (_) {
  var _error = _;
  var _hasError = true;
} finally {
  await babelHelpers.dispose(_stack, _error, _hasError);
}
```

### 装饰器元数据

随着 22 年 3 月的 TC39 会议上，新版装饰器终于迈入到 Stage 3 阶段，TypeScript 在 5.0 版本中也提供了对应于提案的实现。而在 23 年 5 月的 TC39 会议上，装饰器的好朋友元数据对应的提案也进入到了 Stage 3 阶段（需要注意的是，此前我们使用的元数据能力，实际上来自于反射元数据 [reflect-metadata](https://github.com/rbuckton/reflect-metadata) 提案以及它提供的 Polyfill 能力，而本次的 Decorator Metadata 提案则是一个全新的提案，甚至它的仓库都是在装饰器提案迈入到 Stage 3 后创建的）。元数据，即“用于描述数据的数据”，比如在此前使用 reflect-metadata，我们可以这样为一个 Class 内部的属性添加元数据：

```typescript
class User {
  name: string;
}

// 注册元数据
Reflect.defineMetadata('description', '这是 name 属性', User.prototype, 'name');
// 读取元数据
Reflect.getMetadata('description', User.prototype, 'name');
```

为什么我们需要元数据？以上面的例子为例，当我们为 Class 的属性注入了元数据后，就可以通过收集这些元数据来自动生成详细的说明文档，而对原本的运行时没有任何影响。更进一步的，我们可以注入描述数据类型的元数据来实现运行时的额外校验逻辑，注入描述额外操作的元数据来实现面向切面编程，注入描述依赖的元数据来实现依赖注入...等等。而元数据之所以和装饰器有着紧密的关联，最重要的一点就是装饰器能够在满足元数据注册/读取入参的同时，提供更简洁的 API，比如 reflect-metadata 中自带的装饰器 API：

```typescript
class User {
  @Reflect.metadata('description', '这是 name 属性')
  name: string;
}
```

而前后两个元数据提案最明显的差异在于，此前版本的装饰器允许所有类型的装饰器访问被装饰成员所在类的原型，因此装饰器内部可以很方便地将类作为一个 WeakMap 的 key 来存储元数据，而在新版本中，装饰器无法再访问类，而只能访问被装饰的成员。因此此提案提出，在新版装饰器的 context 中，新增一个专用于元数据读写的属性 metadata：

```typescript
type Decorator = (value: Input, context: {
  kind: string;
  name: string | symbol;
  access: {
    get?(): unknown;
    set?(value: unknown): void;
  };
  isPrivate?: boolean;
  isStatic?: boolean;
  addInitializer?(initializer: () => void): void;
  // 新增 metadata 属性
  metadata?: Record<string | number | symbol, unknown>;
}) => Output | void;
```

这种方式使得我们仍然可以使用装饰器来完成元数据的注册：

```javascript
function meta(key, value) {
  return (_, context) => {
    context.metadata[key] = value;
  };
}

@meta('a' 'x')
class C {
  @meta('b', 'y')
  m() {}
}
```

而 context.metadata 这个对象，即 Class 的元数据信息会被合并到类的 \[Symbol.metadata] 属性上：

```javascript
C[Symbol.metadata].a; // 'x'
C[Symbol.metadata].b; // 'y'
```

同时，元数据会随着类的继承一同被继承，但可以通过覆盖被装饰成员来对应覆盖此前元数据：

```javascript
class D extends C {
  @meta('b', 'z')
  m() {}
}

D[Symbol.metadata].a; // 'x' 继承
D[Symbol.metadata].b; // 'z' 覆盖
```

在 5.2 版本中，TypeScript 对应地支持了装饰器元数据提案，实现方式当然也是同样的 context.metadata。如果你希望了解更多关于新版装饰器的使用方式，可以查找此账号发表过的 「TypeScript 5.0 beta 发布：新版 ES 装饰器、泛型参数的常量修饰、枚举增强等」一文。

> Tips：最近笔者在一些交流群里发现部分同学使用装饰器时，遇到了奇怪的错误，比如 Unable to resolve signature of class decorator when called as an expression, The runtime will invoke the decorator with 3 arguments, but the decorator expects 1 这样的错误，其原因就是你的本地 TS 版本已经是 5.0，TS Server 会使用新版的装饰器语法来进行检查，而你本地还是按照旧版装饰器的语法来的。 要解决这个问题，你可以确保本地安装了 4.x 版本的 TS，然后在命令面板中查找「切换 TypeScript 版本」，选择「使用工作区版本」即可。

### 匿名 & 具名元祖元素混用

TypeScript 支持描述一个定长、元素类型固定的数组类型，即元组，如：

```typescript
const arr: [string, string, string, number] = ['lin', 'bu', 'du', 18];
```

TypeScript 4.0 中，支持了为元组的各个元素指定类似属性名的标记以及可选符号([Labeled Tuple Elements](https://github.com/Microsoft/TypeScript/issues/28259)）：

```typescript
const arr: [name: string, age: number, male?: boolean] = ['linbudu', 18, true];
```

但这和此前的匿名元组是互斥的，即要么全部匿名，要么全部具名，不能存在部分匿名部分具名的情况。而在 5.2 版本中，现在元组可以混用匿名元素与具名元素：

```typescript
const arr: [name: string, number, boolean] = ['linbudu', 18, true];
```

### 联合类型数组方法调用

TypeScript 此前对于由数组类型组成的联合类型上的方法调用并不是很智能，比如：

```typescript
declare let array: string[] | number[];

// 此表达式不可调用。
array.filter((x) => !!x);
// 此表达式不可调用。
array.every((x) => !!x);
// 此表达式不可调用。
array.find((x) => !!x);
```

这是因为在这种情况下， TypeScript 希望找到一个同时兼容 string\[] 和 number\[] 的类型来通过 filter 的类型检查（注意，不是 (string | number)\[]），即 filter 的参数类型又是 string 又是 number，但很显然并不存在这样的类型。

而在 5.2 版本中，TypeScript 改进了检查策略，会为这种数组组成的联合类型执行特殊的策略，即使用每个数组的元素类型来构造一个联合类型，然后作为 filter 的参数类型，比如在上面的类型中，filter 的参数类型会是 string | number，同时返回值类型也会对应地变为 Array\<string | number>。

全文完，我们 TS 5.3 beta 见 :-)
