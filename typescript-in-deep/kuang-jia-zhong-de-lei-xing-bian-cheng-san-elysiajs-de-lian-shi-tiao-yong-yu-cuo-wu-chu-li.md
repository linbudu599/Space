# 框架中的类型编程(三)：ElysiaJS 的链式调用与错误处理

这是专栏「框架中的类型编程」的第三篇内容，你可以在 [Framework Typings](https://github.com/linbudu599/framework-typings) 或 [笔者的个人博客](https://linbudu.gitbook.io/spaces) 找到前两篇内容，在之前的内容里我们介绍了 Prisma、tRPC 以及 Hono 中的类型编程，还顺便对 TypeScript 中的「模板字符串类型」进行了展开介绍。如果你已经理解了此前介绍的类型编程，那么在阅读本篇对 ElysiaJS 的介绍时，也不会理解得更快（摊手，毕竟本篇我们简化过的类型编程代码仍然有100+ 行，比之前三个框架的实现加起来都多。

简单介绍下 [ElysiaJS](https://elysiajs.com/) ，它是一个基于 Bun 的服务端框架，号称比 Express 快 21 倍，同时还提供了非常棒的类型安全，你可以在 ElysiaJS 上同时找到 Hono 的路由参数提取（从 `/user/:id` 分析出参数类型为 `{ id: number }`）与 tRPC 的 Schema Validation（使用 `z.object()` 创建输入输出类型 ），还有一些将在这篇文章里介绍的新的东西。

来看一张官网的宣传图片，揣摩揣摩图里都有哪些神奇的黑科技？

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

我们一个个分析下：

* 首先是从字符串到链式调用的转换，比如，这里的 `/user/profile` 转换为了 `api.user.profile` 的链式调用，这里其实也支持 Hono 那样 `/user/profile/:id` 这样的输入，最后也能够将其类型附加到 RequestHandler 的入参类型上。
* 接着是 Schema Validation，这个是我们的老朋友了，`.patch` 方法的第三个参数接收一个 Schema 定义对象，其提取出的类型会被提供给 RequestHandler。
* RequestHandler 的返回值类型到响应类型（即 data）的转换，可以看到这里定义了几个返回错误的情况，在最终的 error 类型里全部得到了完整保留，甚至包括错误码和错误信息的关联。

在这篇文章里，我们会先分别了解这三点背后的类型编程，再将它们组合成最后 ElysiaJS 形状的代码。

### 字符串到链式调用的转换

在 Hono 的那一篇介绍里其实我们已经很熟悉这种「字符串字面量类型」的解析了，当时我们解析的流程是，先从 `/user/:id/:param` 解析出 `['id', 'param']`，然后再使用索引类型访问，`['id', 'param'][number]` 就得到了 `'id' | 'param'` 类型，而这个例子里其实难度也差不多，以 `/system/user/games` 为例，我们最终需要的结构是 `{ system: { user: games: { } } }` 这样的。

首先，转换到元组这一步肯定是少不了的，相比于直接操作字符串字面量类型，操作联合类型元素要更加直观和简单，也就是说，首先要转换成 `['system', 'user', 'games']` 的元组。接着元组转对象类型就好办了，我们实现一个 `ComposeObject` 工具类型，`ComposeObject<'foo', {}>` 的结果会是 `foo: { }`，然后递归调用，即可从元组生成嵌套的对象类型。

第一步，转换到元组，思路和 Hono 中的实现还是一样的，使用 `/{infer Param}/${infer Rest}` 来进行模式匹配即可，这里比较简单，直接看实现就好了：

```typescript
type ParsePathAsTuple<T extends string> =
  T extends `${string}/${infer Param}/${infer Rest}`
    ? [Param, ...ParsePathAsTuple<`/${Rest}`>]
    : T extends `${string}/${infer Param}`
    ? [Param]
    : [];

//  ['system', 'user', 'games'];
type Testing2 = ParsePathAsTuple<'/system/user/games'>;
```

接着来实现 `ComposeObject` 工具类型，不知道你看到上面的时候会不会觉得这个工具类型的作用眼熟——是的，其实它就是 TS 内置的 Record 类型，只不过我们常见的用法是 `Record<string, number>` 酱婶儿的，而现在我们输入的键值只会是单个的字符串字面量。

来试一下递归调用的效果：

```typescript
type Simplify<T> = {
  [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : Simplify<T[K]>
    : T[K];
} & {};

// {
//   foo: {
//     bar: {
//       baz: {
//       }
//     }
//   }
// }
type Testing4 = Simplify<Record<'foo', Record<'bar', Record<'baz', {}>>>>;
```

> 这里的 `Simplify` 类型仅仅用于在 IDE 内提供最终生成的类型结构展示，并没有额外的效果，直接复制粘贴即可。

那么下一步要考虑的就是，如何从元组生成这个调用效果了，很简单，还是递归——

* `ComposeObjectFromTuple` 应该接受两个参数：Tuple 与 Internal，Tuple 是我们上一步生成的元组，Internal 则是我们放置在对象嵌套最深处的结构（别忘了最终使用是 `api.user.profile.fetch`）。
* 通过 `[infer First, ...infer Rest]` 来进行模式匹配，每次出栈一个字面量类型，作为 `Record` 的输入。
* 检查剩余 Rest 的长度：
  * 如果还有值，则重复上一步，构建出 `Record<'foo', Record<'bar', ...>>` 这样的结构。
  * 如果已经遍历完毕，说明这一次的 First 是元组中最后一个字面量类型，此时它的属性类型即是 Internal 。这里需要注意的是，要通过 `Rest['length'] extends 0` 是否成立。

最终实现会是这样的：

```typescript
type ComposeObjectFromTuple<
  Tuple extends string[],
  Internal extends object = {}
> = Tuple extends [infer First, ...infer Rest]
  ? First extends string
    ? Record<
        First,
        Rest extends string[]
          ? Rest['length'] extends 0
            ? Internal
            : ComposeObjectFromTuple<Rest, Internal>
          : Internal
      >
    : {}
  : {};

// {
//   foo: {
//     bar: {
//       baz: {
//         patch: () => Promise<unknown>;
//       }
//     }
//   }
// }
type Testing5 = Simplify<
  ComposeObjectFromTuple<
    ['foo', 'bar', 'baz'],
    {
      patch: () => Promise<unknown>;
    }
  >
>;
```

### Schema Validation

在开始前，你可能注意到了，ElysiaJS 使用的 Schema Builder 是自己导出的 `t`，而不是之前我们已经初步了解的 Zod / ArkType / ... 等工具，其实这并没什么差别，不管是哪个工具，其最终输出的 Schema 上，一定会留一个获取对应 TS 类型的口子，比如 Zod 的话我们可以这么做：

```typescript
import { z } from 'zod';

type InferType<T> = T extends z.ZodType<infer U> ? U : T;
```

回到 ElysiaJS 的 `t`，它留的口子则是 `static` 属性：

```typescript
import { t } from 'elysia';

type UnwrapSchemaType<T extends ReturnType<typeof t.Object>> = T['static'];
```

来测试一下：

```typescript
const schema = t.Object({
  name: t.String(),
  age: t.Number(),
});

// {
//   name: string;
//   age: number;
// }
type Testing1 = UnwrapSchemaType<typeof schema>;
```

那这一步其实到这就差不多了，后面我们只需要简单地留个泛型参数来保存 Schema 类型，然后再 `UnwrapSchemaType` 一下就能获得原始的 TS 类型了。

### RequestHandler 的返回值类型到响应类型

完整的从返回值类型到响应类型的转换其实并不复杂，用一个泛型参数保存推导出的返回值类型，然后解析其中所有的 Error 类型，合并到 `result.error` 中即可，这个在最后的组装环节讲解要更好理解。这一部分我们其实可以关注一个更有趣的东西：预设错误类型。

不知道你是否注意到了，在最开始官网的图里，我们可以直接 `error(418)` 来抛出错误，在 `switch case` 语句中仍然能为 418 这个错误码自动关联上预设的错误信息 `'I am a teapot'`，这又是如何实现的？

如果在类型层面感觉没什么头绪，不妨想想在逻辑层面你会怎么做？

```typescript
const builtInErrorMap = {
  418: 'I am a teapot',
};

const createErrorMessage = (errorCode: number, errorMessage?: string) =>
  builtInErrorMap[errorCode] ?? `HTTP Error ${errorCode}`;
```

那么类型层面不也是一样的吗，定义一个对象类型来保存映射关系，尝试用 ErrorCode 类型去其中匹配，如果匹配到了就返回值，否则就创建一个新的字面量类型即可：

```typescript
type BuiltInErrorMap = {
  418: 'I am a teapot';
};

type FallbackError<Code extends number> = `HTTP Error ${Code}`;

type BuildErrorMessage<Code extends number> = Code extends keyof BuiltInErrorMap
  ? BuiltInErrorMap[Code]
  : FallbackError<Code>;

type Build418Error = BuildErrorMessage<418>; // "I am a teapot"
type Build500Error = BuildErrorMessage<500>; // "HTTP Error 500"
```

### 组装时间

好好好，把几个比较关键的地方都安排明白了，接下来的组装就很简单了。上面几个部分里，我们的实现都是先构造了输入然后进行实现，在组装时，我们要考虑的就是如何从开发者的输入，推导出我们需要的输入。

我们先把最基础的结构定出来，比如需要的泛型参数：

```typescript
export class App<AppStruct = {}> {
  constructor() {}

  public patch<
    Path extends string,
    Handler,
    Schema extends ReturnType<typeof t.Object>,
    AdditionalStruct = unknown
  >(
    path: Path,
    handler: Handler,
    schema: Schema
  ): App<AdditionalStruct & AppStruct> {
    return this;
  }

  public listen(port: number) {
    return this;
  }
}

export function init<Struct extends App>(domain: string) {
  return {} as Struct extends App<infer T> ? T : never;
}
```

这里 `App<AppStruct = {}>` 中的 `AppStruct` 定义，本质上是为了尽可能模拟 ElysiaJS 的 API，同时类似于 tRPC 讲解中在链式调用下不断附加类型信息的方式，通过 `app.patch.get.patch...` 这样连续的定义，我们就能够不断将这个 App 上附加的信息添加到 `AppStruct` 这个泛型参数内。而这里的「信息」，其实就是每个方法的入参与返回值信息。

我们的 patch 方法一共有三个参数：path、handler、schema，来思考思考它们分别会提供哪些类型信息，我们会怎么使用它们？

* path，提供路径的字符串字面量类型，用于转换成链式调用。
* handler，它的入参类型会依赖 schema 的解析后类型，同时它会提供 `api.system.user.games()` 的返回值类型。
* schema，提供解析后的 TS 类型，它会提供 `api.system.user.games()` 的入参类型。

这样我们就知道如何进行下一步了：

* 首先添加一个 UnwrappedSchema 泛型参数
* 定义 Handler 的约束类型，它依赖 UnwrappedSchema 作为输入
* 本次方法调用带来的 App 结构信息，应该来自于对 Path 泛型参数的解析，即对于 `patch('/user/data')`，需要为 App 新增 `{ user: { data: patch () => {} } }` 的结构。

```typescript
type HandlerStruct<UnwrappedSchema> = (input: {
  body: UnwrappedSchema;
  error: any;
}) => unknown;

export class App<AppStruct = {}> {
  constructor() {}

  public patch<
    Path extends string,
    Handler extends HandlerStruct<UnwrappedSchema>,
    Schema extends ReturnType<typeof t.Object>,
    UnwrappedSchema = UnwrapSchemaType<Schema>,
    AdditionalStruct = ComposeObjectFromTuple<
      ParsePathAsTuple<Path>,
      {
        patch: (body: UnwrappedSchema) => any;
      }
    >
  >(
    path: Path,
    handler: Handler,
    schema: Schema
  ): App<AdditionalStruct & AppStruct> {
    return this;
  }

  public listen(port: number) {
    return this;
  }
}

// 仅仅是提取出 AppStruct
export function init<Struct extends App>(domain: string) {
  return {} as Struct extends App<infer T> ? T : never;
}
```

到这里，其实我们已经实现了两个重要能力：链式调用转换 & Schema Validation，来试试效果：

```typescript
import { App, init } from './Elysia';

const app = new App()
  .patch(
    '/user/profile',
    ({ body, error }) => {
      if (body.age < 18) return error(400, 'Oh no');

      if (body.name === 'Nagisa') return error(418);

      if (body.age === 25) return error(599);

      return body;
    },
    t.Object({
      name: t.String(),
      age: t.Number(),
    })
  )
  .listen(80);

const server = init<typeof app>('localhost');

const res = await server.user.profile.patch({
  name: 'saltyaom',
  age: '21', // x 不能将类型“string”分配给类型“number”。
});
```

下一步我们要思考的是，如何将 Handler 的返回值类型传递给 patch res ，首先明确下，要实现官网图中判断完 `if(res.error)`，并且将所有 `res.error.status` 枚举完毕后，`res.data` 自动获得正确类型的效果，最终的 patch res 类型需要是啥样的？

其实这里甚至不需要上 XOR 这样的互斥类型操作，只需要简单的联合类型即可：`{ data: null; error: true; } | { data: ...; error: null; }`，TS 已经足够聪明啦。

首先，我们肯定要在作为 Internal 参数的 patch 方法的返回值上做文章。其次，data 的有效类型仍然就是 UnwrappedSchema，而 error 的类型其实就是 Handler 的返回值类型（必然是个联合类型）中，符合 Error 结构的那几个类型的组合。所以我们现在可以这么修改下代码，首先定义下 Error 结构，之前 HandlerStruct 中的 error 入参类型可是 any，现在要给它一个名分了，这部分我们在前面已经了解过，直接把代码都复制过来即可：

```typescript
type BuiltInErrorMap = {
  418: 'I am a teapot';
};

type WithError<TErrorCodes extends number, TMessage extends string> = {
  error: {
    status: TErrorCodes;
    value: TMessage;
  };
};

type FallbackError<Code extends number> = `HTTP Error ${Code}`;

type BuildErrorMessage<Code extends number> = Code extends keyof BuiltInErrorMap
  ? BuiltInErrorMap[Code]
  : FallbackError<Code>;

type DefineError = <
  Code extends number,
  Message extends string = BuildErrorMessage<Code>
>(
  errorCode: Code,
  message?: Message
) => WithError<Code, Message>;

type HandlerStruct<UnwrappedSchema> = (input: {
  body: UnwrappedSchema;
  error: DefineError;
}) => unknown;
```

> 一个小细节是，虽然我们为 Handler 定义了 HandlerStruct 作为约束，但由于我们并没有在 HandlerStruct 中定义具体的返回值类型，所以仍然可以通过 `ReturnType<Handler>` 来获取到开发者实际定义的返回值类型。

紧接着，我们现在可以创建一个 TransformHandlerResult 工具类型作为 patch 方法的返回值类型，它接收 UnwrappedSchema、 `ReturnType<Handler>` 作为输入，功能就是组装出需要的 data 与 error 互斥的联合类型：

```typescript
type ExtractErrors<Input> = Input extends { error: infer T } ? T : never;

type TransformHandlerResult<
  UnwrappedSchema,
  HandlerReturns extends ReturnType<HandlerStruct<UnwrappedSchema>>
> = Promise<
  | {
      data: UnwrappedSchema;
      error: null;
    }
  | {
      data: null;
      error: ExtractErrors<HandlerReturns>;
    }
>;
```

这里的一个小技巧是，ExtractErrors 工具类型利用了条件类型的分布式特性，它会将 HandlerReturns 这个联合类型的每一个类型成员都与 `{ error: infer T }` 进行一次匹配，所有符合条件的结构会重新组成联合类型，也就是我们需要的，错误类型的联合啦。

再看一遍完整的代码：

```typescript
import { t } from 'elysia';

type Simplify<T> = {
  [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]
      : Simplify<T[K]>
    : T[K];
} & {};

type UnwrapSchemaType<T extends ReturnType<typeof t.Object>> = Simplify<
  T['static']
>;

type HandlerStruct<UnwrappedSchema> = (input: {
  body: UnwrappedSchema;
  error: DefineError;
}) => unknown;

type BuiltInErrorMap = {
  418: 'I am a teapot';
};

type WithError<TErrorCodes extends number, TMessage extends string> = {
  error: {
    status: TErrorCodes;
    value: TMessage;
  };
};

type FallbackError<Code extends number> = `HTTP Error ${Code}`;

type BuildErrorMessage<Code extends number> = Code extends keyof BuiltInErrorMap
  ? BuiltInErrorMap[Code]
  : FallbackError<Code>;

type DefineError = <
  Code extends number,
  Message extends string = BuildErrorMessage<Code>
>(
  errorCode: Code,
  message?: Message
) => WithError<Code, Message>;

type ParsePathAsTuple<T extends string> =
  T extends `${string}/${infer Param}/${infer Rest}`
    ? [Param, ...ParsePathAsTuple<`/${Rest}`>]
    : T extends `${string}/${infer Param}`
    ? [Param]
    : [];

type ComposeObjectFromTuple<
  T extends string[],
  DeepestStruct extends object = {}
> = T extends [infer First, ...infer Rest]
  ? First extends string
    ? Record<
        First,
        Rest extends string[]
          ? Rest['length'] extends 0
            ? DeepestStruct
            : ComposeObjectFromTuple<Rest, DeepestStruct>
          : DeepestStruct
      >
    : {}
  : {};

type ExtractErrors<Input> = Input extends { error: infer T } ? T : never;

type TransformHandlerResult<
  UnwrappedSchema,
  HandlerReturns extends ReturnType<HandlerStruct<UnwrappedSchema>>
> = Promise<
  | {
      data: UnwrappedSchema;
      error: null;
    }
  | {
      data: null;
      error: ExtractErrors<HandlerReturns>;
    }
>;

export class App<AppStruct = {}> {
  constructor() {}

  public patch<
    Path extends string,
    Handler extends HandlerStruct<UnwrappedSchema>,
    Schema extends ReturnType<typeof t.Object>,
    UnwrappedSchema = UnwrapSchemaType<Schema>,
    AdditionalStruct = ComposeObjectFromTuple<
      ParsePathAsTuple<Path>,
      {
        patch: (
          body: UnwrappedSchema
        ) => TransformHandlerResult<UnwrappedSchema, ReturnType<Handler>>;
      }
    >
  >(
    path: Path,
    handler: Handler,
    schema: Schema
  ): App<AdditionalStruct & AppStruct> {
    return this;
  }

  public listen(port: number) {
    return this;
  }
}

export function init<Struct extends App>(domain: string) {
  return {} as Struct extends App<infer T> ? T : never;
}
```

现在再来试试效果：

```typescript
const res = await server.user.profile.patch({
  name: 'saltyaom',
  age: '21', // x
});

if (res.error) {
  switch (res.error.status) {
    case 400:
      throw res.error.value; // "Oh no"
    case 418:
      throw res.error.value; // "I am a teapot"
    case 599:
      throw res.error.value; // "HTTP Error 599"
  }
}

res.data.name.toString(); // √
```

搞定！

### 总结一下

这次的 ElysiaJs 的类型编程实现确实有点复杂，但实际上实现过程还是一样的步骤：明确泛型信息的提供方，预设最终类型的结构，然后就是编写工具类型，让输入一步步变成输出的形状，就这么简单。如果本节的内容你认为理解起来有些困难，不妨将我们的各个拼装部分再拆分成更细粒度的实现。

我个人认为，本节的示例非常适合作为类型编程的综合练习题，毕竟它用到了泛型、联合类型、模板字符串类型、映射类型、条件类型等等等等一堆概念，如果能真正把这些概念都融汇贯通，那 TypeScript 简直是想学不会都难 ^\_^
