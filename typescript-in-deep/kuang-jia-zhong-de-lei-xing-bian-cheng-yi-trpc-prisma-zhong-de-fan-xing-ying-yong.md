---
description: 2024-1-28
---

# 框架中的类型编程(一)：tRPC & Prisma 中的泛型应用

最开始想写框架中的类型编程这个题材，是因为突然看到了 Hono 这么个服务端框架，它的特色之一就是能分析请求的路径来提供参数的类型，比如定义 `/:name/:page` 的路径，在你调用 `param()` 方法时 Hono 能够自动提示出 `'name' | 'page'` 这样的联合类型。

我不确定 Hono 是不是第一个这么做的服务端框架，毕竟我知道有这么个框架的时候它已经 10k star 了。同时我还看到了一个使用 Hono + Prisma 的示例，两年前我的 TypeScript 还不咋地的时候就知道 Prisma 有很棒的类型提示（《Prisma：下一代ORM，不仅仅是ORM》，[上篇](https://juejin.cn/post/6973277530996342798)，[下篇](https://juejin.cn/post/6973950142445518884)），因此我就想到了，是不是可以整一篇简单的文章介绍下，这些工具中的类型编程是如何实现的？

这个题材值得花费精力的原因在于，它为刚入门 TypeScript 的同学展示了 TypeScript 的无限可能性（或者说自己还有多少要学的），也让已经有一定 TypeScript 编写经验的同学意识到，其实自己此前还有很多技巧没有用上。同时，这个题材中选择的框架，其展示出来的类型编程都是非常接地气的，个中技巧完全可以掰开理解化为己用。

始料未及的是，大纲时的小作文，最后变成了接近 9k 字的上下两篇，而且我感觉这完全可以做成一个新的专栏，名字就叫「框架中的类型编程」:-)。

在已有的上下两篇中，我们会介绍四个类型编程的场景：tRPC、Hono、Prisma 以及 EventEmitter。难度可以排列为 tRPC < EventEmitter < Prisma ≈ Hono，这几个例子中的概念实现相对独立，你可以根据自己的喜好以任意的顺序阅读。

### tRPC: 链式调用中的泛型补全

tRPC 是一个基于 TypeScript 的 RPC 框架（怎么感觉说了好像没说），它的核心优势之一是在前后端提供统一的类型定义，想象一下，之前你请求服务端接口时，对它到底会返回个什么东西根本猜不到，最多自己手写一套类型把请求结果断言过去：

```ts
const result = await axios<PostResponse>('/api/post');
```

而使用 tRPC，请求其实就是调用一个已经拥有强类型的函数：

```ts
const postQuery = trpc.post.useQuery({ id });
```

在 useQuery 函数中，其入参与出参都具有完整的类型约束，而它实际上是通过这么一个 pipeline 的方式来生成的（tRPC 中叫 procedure）：

```ts
import { initTRPC } from '@trpc/server';
import { z } from 'zod';
import { createTRPCReact } from '@trpc/react-query';

export const t = initTRPC.create();
export const appRouter = t.router({
  post: t.procedure
    .input(
      z.object({
        id: z.number(),
      })
    )
    .query(async (input) => {
      // 拥有 number 类型！
      const id = input.id;
      const result = await prisma.posts.findUnique({
        where: { id },
      });
      
      return result;
    }),
});
```

以上代码的核心在于，通过 `.input` 方法 + Zod 的 Schema 定义，可以为接下来链式调用的 query 方法提供入参信息！还记得很早以前，第一次看到在链式调用中提供类型信息的时候，我其实还震惊了挺久，感觉怎么别人脑瓜子就这么好用呢。

但其实你琢磨一下，链式调用通常是通过 Class + 每个方法再次返回实例来实现的，而每个方法返回的实例，虽然值并不会发生变化，但上面的类型是可以发生变化的。因此我们的实现思路大致是酱紫的：

* 在 Class 上声明数个泛型。
* 方法调用时，使用从该次调用获得的类型，填充实例上的一部分泛型，再返回这个填充完毕的实例类型。
* 后续的方法直接依赖实例上的类型，这样就实现了链式调用的泛型填充效果。

我们从最简单的例子开始，首先定义一个只有一个泛型坑位的 Class，以及一个 input 方法，它也拥有一个泛型，其信息来源于自己的入参：

```ts
export class Pipeline<PipelineInputType = any> {
  input<InputType>(input: InputType) {
    return this as any;
  }
}

// 泛型信息填充为 string
new Pipeline().input('linbudu');

// 泛型信息填充为 { name: string }
new Pipeline().input({
  name: 'linbudu',
});
```

在 input 调用获得这个泛型信息后，直接返回一个 `Pipeline<InputType>` 类型，这样后续进行的链式调用，都可以直接引用 Class 上的 PipelineInputType 类型了：

```ts
export class Pipeline<PipelineInputType = any> {
  input<InputType>(input: InputType): Pipeline<InputType> {
    return this as any;
  }

  query(callback: (options: { input: PipelineInputType }) => any): void {}
}

new Pipeline().input('linbudu').query(({ input }) => {
  // string 类型！
  input.startsWith('');
});

new Pipeline()
  .input({
    name: 'linbudu',
  })
  .query(({ input }) => {
    // { name: string; } 类型！
    input.name.startsWith('');
  });
```

到这里其实最重要的第一步就已经完成了，tRPC 中的这部分类型编程其实关键点就在于\*\*「在链式调用中填充类型信息」\*\*，但这个例子里只填充了一次多少显得不够筋道，我们还可以完善一点，就像 tRPC 中不仅可以通过 `.input` 定义 query 的入参类型，还可以通过 `.output` 约束 query 的出参类型。而你都完成了第一步，那这里其实也很简单了，不就是多一个泛型坑位的事吗？

```ts
export class Pipeline<PipelineInputType = any, PipelineOuputType = any> {
  input<InputType>(input: InputType): Pipeline<InputType, PipelineOuputType> {
    return this as any;
  }

  output<OuputType>(input: OuputType): Pipeline<PipelineInputType, OuputType> {
    return this as any;
  }

  query(
    callback: (options: { input: PipelineInputType }) => PipelineOuputType
  ): void {}
}

new Pipeline()
  .input('linbudu')
  .output(true)
  .query(({ input }) => {
    //  不能将类型“string”分配给类型“boolean”。
    return input;
  });
```

看起来不错！但是这里有个很奇怪的地方，我们是用一个值来作为类型约束的（`.input('linbudu')`、`.output(true)`），这非常的不规范，而 tRPC 的示例是通过 zod 来进行类型约束的，看起来就正经很多，而且也可以直接作为校验器进行验证。

```ts
.input(z.string())
.output(z.boolean())
```

至于 zod 、yup、superstruct、arktype...这些工具，其实大差不差，核心功能都是在创建一个 Validator 的同时提供与 Schema 一致的类型提示。但需要注意的是，`z.string()` 返回的并不是一个原始的 string 类型，而是 zod 中的 ZodString 类型，上面提供了如 `z.string().regex()`、`z.string().datetime()` 这样的方法来实现命令式的校验。

那你是不是就会开始发愁，怎么从校验库的内置类型获得原始的 TypeScript 类型呢？其实，这些内置类型上都保留了一份原始类型——不仅仅是 zod，所有号称 First Class TypeScript Support 的校验库都是这么做的。在 zod 中，这个「后门」是 `_output` 属性：

```ts
z.string()['_output']; // string
z.number()['_output']; // string
z.object({
  name: z.string(),
})['_output']; // { name: string }
```

另外，这些校验库其实也提供了内置的工具类型来提取原始类型，比如 zod 中的 ZodType 就是，我们可以实现一个类型来提取原始类型：

```typescript
type InferType<T> = T extends z.ZodType<infer U> ? U : T;

type _ = InferType<ReturnType<typeof z.string>>; // string
// or
// type _ = InferType<z.ZodString>; // string
```

先假设我们只会使用 zod 来定义 Schema，现在我们要考虑的是，如何实现提取出 ZodString 类型，却使用 string 类型来填充泛型的效果。这其实就是类型编程中最经典最基础的操作，**很多时候泛型第一次获得的类型无法直接利用，需要经过数次加工后才能传递给下一个泛型类型的消费方。**

现在我们定义 InferredInputType 、InferredOutputType 这两个泛型参数，至于 input / output 方法中用于获得 ZodType 的泛型参数，并不需要占用 Pipeline 上的泛型坑位：

```typescript
import { z } from 'zod';

type InferType<T> = T extends z.ZodType<infer U> ? U : T;

export class Pipeline<InferredInputType = any, InferredOutputType = any> {
  input<InputType extends any>(
    input: InputType
  ): Omit<Pipeline<InferType<InputType>, InferredOutputType>, 'input'> {
    return this as any;
  }

  ouput<OuputType>(
    input: OuputType
  ): Omit<Pipeline<InferredInputType, InferType<OuputType>>, 'output'> {
    return this as any;
  }

  // query 完全是在消费泛型，不会产生新的泛型或更新泛型，所以不需要声明泛型
  query(
    callback: (options: { input: InferredInputType }) => InferredOutputType
  ): void {}
}

```

完整版的最终代码还多了一些东西，比如返回值类型现在是 `Omit<Pipeline, 'input'>` 这样的结构，原因在于我们希望每个方法调用在实例上只会调用一次，而不是 `.input().input().input()` 这样，因此可以用这种方式来从实例上移除当前被调用的方法。

如果上面的例子没有榨干你的 CPU，你很容易产出这样的疑问：如果我不只是在使用 zod 呢？刚刚上面提了那么多的校验库，如果我就是看别的更顺眼呢？tRPC 支持那么多校验库，它又是怎么实现的？

原理很简单——加机器。tRPC 中内置了对这些校验库的类型提取逻辑，校验库通常都会有一个 validate 方法（也有的叫 parse，create 之类的），这些方法的返回值也是原始类型，反正我就条件类型枚举呗，不是这个就是那个，总有一个分支能把你逮住咯，请看 [tRPC 源码](https://github.com/trpc/trpc/blob/602bca563bad7fc999bfea303263d6b4ca1d9c54/packages/server/src/unstable-core-do-not-import/parser.ts#L2)。

总结一下，tRPC 这个部分的类型编程还是比较简单的，主要就是「链式调用 x 泛型传递」这里比较绕一点，因此把它作为第一个场景，让你在冬日里停滞的大脑稍稍 warm up 一下。

### Prisma: 泛型类型的流转

Prisma 是一个比较特殊的 ORM，特殊就特殊在它不像 TypeORM、Sequelize 这样是通过 TS 代码来定义数据库表结构，而是通过自己的 DSL：Prisma Schema 来定义结构的。Prisma Schema 会被映射到 TypeScript 类型，并通过内置的大量类型编程来实现严格的类型约束，比如以下的 Prisma Schema：

```
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}
```

会编译出这样的 User 类型：

```typescript
type User = {
    id: number;
    email: string;
    name: string | null;
}
```

以及消费这个 User 类型的各种方法，以 findUnique 为例：

```ts
const prisma = new PrismaClient();

async function main() {
  const queryUserRes = await prisma.user.findUnique({
    select: {
      id: true,
      email: true,
    },
    where: {
      id: 599,
    },
  });

  queryUserRes?.id; // number
  queryUserRes?.email; // string
  queryUserRes?.name; // 不存在此属性
}
```

在上面的例子中，queryUserRes 的类型完全由 findUnique 方法中的 select 属性来决定——你 select 哪些属性，哪些属性就会出现在结果中，是不是很神奇？

这一节我们要实现的就是这么一个效果，简化后的代码大致是这样的：

```typescript
declare function findUnique(params: ?): ?;

const res = findUnique({
  where: {
    id: 1,
  },
  select: {
    id: true,
    email: false,
  },
});

res.id; // √
res.email; // x
```

如果你仔细读完了上一节的内容，那么多少会有一点思路，我们要做的还是泛型信息的流转变换，所以事前分析就相当有必要了：

* 首先，泛型信息由 select 属性的值提供，等价于其字面量类型。这个可以直接通过 extends 实现。
* 接着，从字面量类型中提取由类型为 true 的属性组成的部分，比如 `{ id: true; name: false; }` 的结果是 `{ id: true; }`，这个可以通过 Pick + 索引类型 + 条件类型来实现。
* 使用提取出的属性名，去 User 类型中匹配对应的类型。这个可以通过取交集来实现。

我们从第一步开始一步步完善这个方法，首先 findUnique 方法的参数会是个对象类型，属性包括 where 与 select：

```ts
type User = {
  id: number;
  email: string;
  name: string | null;
};

interface FindUniqueParams<SelectFields> {
  where: Partial<User>;
  select: SelectFields;
}
```

而 FindUniqueParams 接口的 SelectFields 泛型，将来自于调用 findUnique 方法时的输入：

```ts
declare function findUnique<SelectFields>(
  params: FindUniqueParams<SelectFields>
): SelectFields | undefined;
```

`FindUniqueParams<SelectFields>` 可能不太好理解，来看实际的例子：

```ts
const res = findUnique({
  where: {
    id: 1,
  },
  select: {
    id: true,
  },
});
```

在这里，整个 params 是 `FindUniqueParams<SelectFields>` 类型，那么有以下的等式：

```ts
{
  where: { id: number };
  select: { id: boolean }
}
// 等价于
{
  where: Partial<User>;
  select: SelectFields;
}
```

那么 SelectFields 不就是 `{ id: boolean }` 类型？

这个时候，其实返回值 res 的类型就是 `{ id: boolean }`。接下来，我们可以取 `{ id: boolean }` 与 User 类型的交集，即实现一个工具类型 `ObjectIntersection<A, B>`，能够提取 A 与 B 这两个对象类型之间共同存在的属性，如果属性类型在两个对象类型中不一致，则以 B 为准。

这个时候我就要掏出一张两年前做的图了：

> 还有什么看着比较高大上的画图工具，请务必推荐给我...

![](https://s3.bmp.ovh/imgs/2024/01/27/cc3b39829a545548.png)

TypeScript 内置的工具类型 Exclude 与 Extract，也可以用更好理解的方式称之为差集 Difference 与交集 Intersection，这里我们主要介绍交集：

```typescript
type Extract<T, U> = T extends U ? T : never;

type AExtractB = Extract<1 | 2 | 3, 1 | 2 | 4>; // 1 | 2
```

之所以会有这种「集合」的表现，实际上得益于条件类型的分布式特性，如上面的例子实际上等价于：

```ts
type _AExtractB =
  | (1 extends 1 | 2 | 4 ? 1 : never) // 1
  | (2 extends 1 | 2 | 4 ? 2 : never) // 2
  | (3 extends 1 | 2 | 4 ? 3 : never); // never
```

> 关于集合工具类型，真的能展开讲解非常多，分布式条件类型，一维，二维，二维的前序优先，二维的后序优先，blabla...，这里我们先收一下吧。

这里我们需要的也就是交集类型，但是要「稍微」扩展一下，因为我们需要取的是两个对象的交集，想想怎么转换到一维的交集？对象=键值对，那两个对象的键取交集，然后再在希望具有更高优先级属性类型的那个对象里 Pick 一下就好啦\~

```ts
export type PlainObjectType = Record<string, any>;

// 一维集合的交集实现
export type Intersection<A, B> = A extends B ? A : never;

// 对象键集合的交集实现
export type ObjectKeysIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Intersection<keyof T, keyof U>;

// 对象的交集实现！
export type ObjectIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Pick<T, ObjectKeysIntersection<T, U>>;
```

再改写一下 findUnique 的签名，在返回值里进行交集运算：

```ts
declare function findUnique<SelectFields extends PlainObjectType>(
  params: FindUniqueParams<SelectFields>
): ObjectIntersection<User, SelectFields> | undefined;
```

你会发现现在已经可以检查出没有 select 的属性了：

![](https://s3.bmp.ovh/imgs/2024/01/27/b1f7348c5ec661dd.png)

然而，如果添加个 `email: false`，你会发现这种情况下错误也消失了：

![](https://s3.bmp.ovh/imgs/2024/01/27/b5bad24001364225.png)

同时，在这么做的过程中你还会感觉到一丝不对劲：select 应该只能出现 User 中存在的属性才对，所以应该是有类型提示的，另外属性也只能是 boolean 类型，现在却是来者不拒的状态？

我们继续完善实现，首先我们应该先给 SelectFields 提供类型约束，它只能是 User 中的属性 + boolean 才对：

```typescript
type UserSelectConstraint = Partial<Record<keyof User, boolean>>;
```

把这个约束添加给所有引用 SelectFields 的地方：

```typescript

interface FindUniqueParams<SelectFields extends UserSelectConstraint> {
  where: Partial<User>;
  select: SelectFields;
}

declare function findUnique<SelectFields extends UserSelectConstraint>(
  params: FindUniqueParams<SelectFields>
): ObjectIntersection<User, SelectFields> | undefined;
```

这样一来我们就获得了类型提示，并且也不再允许非 boolean 类型的值：

> 你可能会发现这里还是允许非 User 的属性，这个我们最后说。

![](https://s3.bmp.ovh/imgs/2024/01/27/cee39ad131d14746.png)

下一个问题，即使选择的属性值为 false，我们最终的结果里也还是会有这个属性？

这里还是可以用集合的思路解决：对于提取的 SelectFields 属性，我们提取其属性类型为 true 的子集即可，来实现一个 PickByValue 类型：

```ts
export type PickByValueType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K];
};
```

> 简单说明一下这个类型的实现：
>
> * 遍历类型 T 的属性，在重映射中（`as` 语法）过滤属性。
> * 如果属性的值类型满足 ValueType，则在重映射中返回这个属性。
> * 否则，返回 never，则此属性会消失在结果的类型中。

最后再完善下 findUnique 的类型定义：

```ts
declare function findUnique<SelectFields extends UserSelectConstraint>(
  params: FindUniqueParams<SelectFields>
): ObjectIntersection<User, PickByValueType<SelectFields, true>> | undefined;
```

![](https://s3.bmp.ovh/imgs/2024/01/27/55e66177f91d7172.png)

完整的代码如下：

```ts
type User = {
  id: number;
  email: string;
  name: string | null;
};

type UserSelectConstraint = Partial<Record<keyof User, boolean>>;

interface FindUniqueParams<SelectFields extends UserSelectConstraint> {
  where: Partial<User>;
  select: SelectFields;
}

export type PlainObjectType = Record<string, any>;

export type Intersection<A, B> = A extends B ? A : never;

export type ObjectKeysIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Intersection<keyof T, keyof U>;

export type ObjectIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Pick<T, ObjectKeysIntersection<T, U>>;

export type PickByValueType<T, ValueType> = {
  [K in keyof T as T[K] extends ValueType ? K : never]: T[K];
};

declare function findUnique<SelectFields extends UserSelectConstraint>(
  params: FindUniqueParams<SelectFields>
): ObjectIntersection<User, PickByValueType<SelectFields, true>> | undefined;
```

最后来解释一下上面我们的 select 为何允许非 User 的属性，这是因为在 Prisma 中，select 的类型定义同样是从 Schema 生成的，它是一个已经明确的对象类型，而非我们上面从输入各种推导转换得到的。

总结一下，Prisma 这个示例比 tRPC 中稍微绕一些，但只要提前分析好泛型信息的流转，确认输入泛型如何转换能得到预期的结果，就没那么复杂啦。另外，强烈建议挑几个别的 Prisma 生成的方法来自己试一下从头实现，类型编程就跟五年高考三年模拟一样，光看解析是永远学不会的。
