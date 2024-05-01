---
description: 2024-1-28
---

# 框架中的类型编程(二)： Hono  中的模板字符串类型编程

### Hono: 模板字符串类型的妙用

Hono 是一个极简的服务端框架，其优势主要在于 Edge 场景下的原生支持（Vercel Functions，AWS Lambda，Deno，Bun 等）以及超棒的 TypeScript 研发体验。后者也是它会出现在这篇文章里的原因，Hono 能够从请求的路径中提取参数，并作为类型约束：

```ts
app.get('/user/:name/:id', (c) => {
  const name = c.req.param('name'); // √
  const level = c.req.param('level'); // x
  ...
})
```

在上面的代码中，你定义 `/:name/:id` 的路径后，Hono 能够解析出你的请求中存在 name 与 id 参数，`c.req.param` 被推导为一个仅接受 `'name' | 'id'` 联合类型参数的函数，在这一节我们要实现的就是这么个效果。

这部分的内容需要你对 TypeScript 中的模板字符串类型有基本的了解，如果你此前完全不理解或是知之甚少，不妨先拉到文章的最下方补补课再回来，我会一直在这里等你 :-)

首先我们会对整体代码进行简化，最终的代码结构大致会是这样：

```ts
export declare function request(
  requestPath: ?,
  handler: ?
): void;

request('/:name', (ctx) => {
  ctx.param('name'); // √
  ctx.param('id'); // x
});

request('/:name/:id', (ctx) => {
  ctx.param('name'); // √
  ctx.param('id'); // √
  ctx.param('level'); // x
});
```

接着，我们先来分析下泛型信息的流转，这也是在上一部分中我们着重强调的思考方式。

* 类型的来源一定是第一个参数 requestPath，它一定是个字符串，得到的类型也是字符串字面量类型。
* 在获得字面量类型后，我们要对其进行解析，提取出其中的参数部分，这里我们需要用到模板字符串类型的模式匹配。
* 提取出参数部分之后，要再把参数信息转换为联合类型的形式，作为 handler 的参数类型，来实现类型约束的效果。

先是前两步，可以直接看代码：

```ts
export declare function request<Path extends string>(
  requestPath: Path,
  handler: () => void
): void;
```

![](https://s3.bmp.ovh/imgs/2024/01/27/be7dc92918501d9d.png)

然后来实现 handler 的类型，它肯定会是一个单独的工具类型，并接受 Path 这个类型参数：

```ts
type RequestHandler<Path extends string> = (ctx: {
  param(key: ?): string;
}) => void;

export declare function request<Path extends string>(
  requestPath: Path,
  handler: RequestHandler<Path>
): void;
```

RequestHandler 中的 Path 就是我们得到的 `/:name` 字面量类型，这里和此前略有不同，我们是先把泛型信息原样交给另一个工具类型，再由它内部自行处理的。那么唯一的问题就是，如何将 `/:name` 类型转换为 `'name'` 类型？

只要涉及模板字符串类型的转换，基本可以奔着模式匹配去，这里也是一样。我们先只考虑路径只会是 `/:name` 、`/:id` 这样简单结构的情况：

```ts
type ExtractParam<T extends string> = T extends `${string}/:${infer Param}`
  ? Param
  : '';

type ExtractParam_Res_1 = ExtractParam<'/users/:id'>; // id
type ExtractParam_Res_2 = ExtractParam<'/users/'>; // ''
```

简单解析一下：

* 由于我们不使用 `/users/:id` 中的 `/users`，在匹配时直接使用 `${string}` 而不是 `${infer Prefix}` 的方式，减少一个无意义的变量声明。
* 如果字符串满足 `/$prefix/:$param` 的结构，就返回 param 的字面量类型。

很好，这个时候我们就完成了最重要的一步转换：「请求路径」到「参数类型」的提取：

```ts
type ExtractParam<T extends string> = T extends `${string}/:${infer Param}`
  ? Param
  : '';

type RequestHandler<Path extends string> = (ctx: {
  param(key: ExtractParam<Path>): string;
}) => void;

export declare function request<Path extends string>(
  requestPath: Path,
  handler: RequestHandler<Path>
): void;
```

![](https://s3.bmp.ovh/imgs/2024/01/27/f58a8a532173c1bd.png)

到此为止也太寒碜了点，我们现在的解析逻辑只能解析单个 param 的固定结构，如果多来几个，比如 `/:param/:id` 这样，那就会导致解析逻辑异常（解析出的参数名会变成 `'param/:id'`）。

要解决这个问题也好办，只需要让我们的 ExtractParam 类型能够解析多个参数，再将这些参数转换为联合类型即可。

而如果模板字符串类型需要解析一个类型中的多个参数，那直奔着递归去就行了。对前面的模式匹配进行改写，现在我们去匹配 `$prefix/:$param/$rest` 这样的结构，然后对 `$rest` 进行递归地匹配：

```ts
type ExtractMuliParams<T extends string> =
  T extends `${string}/:${infer Param}/${infer Rest}`
    ? [Param, ...ExtractMuliParams<`/${Rest}`>]
    : T extends `${string}/:${infer Param}`
    ? [Param]
    : [];

type ExtractMuliParams_Res_1 = ExtractMuliParams<'/users/:id'>; // ['id']
type ExtractMuliParams_Res_2 = ExtractMuliParams<'/users/:id/:name'>; // ['id', 'name']
type ExtractMuliParams_Res_3 = ExtractMuliParams<'/users/:id/:name/profile'>; // ['id', 'name']
type ExtractMuliParams_Res_4 = ExtractMuliParams<'/users/:id/:name/profile/'>; // ['id', 'name']
```

至于怎么把 `['id', 'name']` 转换为 `'name' | 'id'`，这个就比较简单了，`Params[number]` 即可。

最终的代码会是酱婶儿的：

```ts
type ExtractMuliParams<T extends string> =
  T extends `${string}/:${infer Param}/${infer Rest}`
    ? [Param, ...ExtractMuliParams<`/${Rest}`>]
    : T extends `${string}/:${infer Param}`
    ? [Param]
    : [];

type Context<Path extends string> = {
  param(key: ExtractMuliParams<Path>[number]): string;
};

type RequestHandler<Path extends string> = (ctx: Context<Path>) => void;

export declare function request<Path extends string>(
  requestPath: Path,
  handler: RequestHandler<Path>
): void;
```

整理一下泛型信息的流转：

* requestPath 参数提供了 Path 参数类型，被 RequestHandler → Context 消费。
* Context 中，会调用 ExtractMuliParams 工具类型，先将 Path 解析为一个字符串数组，再使用 `[number]` 语法得到字符串联合类型。

最终的效果也相当还原：

![](https://s3.bmp.ovh/imgs/2024/01/27/62397a6d066783ce.png)

### 模板字符串类型扩展：EventEmitter

在上面的例子中我们介绍的是模板字符串类型从「输入的字符串值」到「字符串字面量类型」的转换，这是在实际应用中较为少见的部分——因为框架的开发者通常已经为你准备好了。而更常见，也更简单的部分，实际上是「对字符串字面量类型进行类型编程」这么一个部分。

基于模板字符串类型的自动分发特性（如果不知道是啥，请看下方快速入门），我们只需要声明一组字面量联合类型，就可以通过这个特性让联合类型变成我们想要的形状，实际应用中一个常见的例子就是事件监听器。

假设我们有这么一组联合类型，表达了一个 VideoPlayer 上支持的事件：

```ts
type Events = 'play' | 'pause' | 'restart' | 'update' | 'close' | 'block-video';
```

现在我们要开发一个 React 组件来封装 VideoPlayer，它的属性中会包括直接对这些事件的监听，但需要经过命名的转换，如 `onPlay`、`onUpdate`、`onBlockVideo` 。如果只有少数几个事件，一个个手写看起来也没什么，但身为一名程序员，肯定要想方设法把已经有的事件联合类型给利用起来，这样后续迭代中，只需要增删联合类型的事件，就能够自动在属性中生效。

首先我们要解决的是，如何将这个联合类型转换为 `'onPlay' | 'onPause' | 'onRestart'` 这样的结构，这一步看起来还比较简单，只需要把事件类型首字符大写，前面加个 on 就好了：

```ts
type ToEventHandler<TEvent extends string> = `on${Capitalize<TEvent>}`;
```

来看看效果：

![](https://s3.bmp.ovh/imgs/2024/01/27/cbf5200a0c20aaab.png)

看起来似乎不太对，我们没有处理 KebabCase 的事件类型...。但其实也并不复杂，这里我们能够确定字符串的转换规则，即将字符串 `foo-bar-baz` 按 `-` 拆分后，将 `bar`、`baz` 转换为首字符大写即可，写一个 `KebabCase2CamelCase` 类型：

```ts
type KebabCase2CamelCase<S extends string> =
  S extends `${infer Head}${'-'}${infer Rest}`
    ? `${Head}${KebabCase2CamelCase<Capitalize<Rest>>}`
    : S;
```

简单分析一下，首先尝试对字符串应用 `Head-Rest` 的模式匹配，如果成功应用，保持 Head 不动，对 Rest 应用 `Capitalize`，而 Rest 中可能还存在 `Head-Rest` 的模式匹配，因此要递归调用这个工具类型，直到再也无法匹配为止。

然后用这个类型来完善下之前的 ToEventHandler 类型：

```ts
type ToEventHandler<TEvent extends string> = `on${Capitalize<
  KebabCase2CamelCase<TEvent>
>}`;
```

> 这里我们实现的 `KebabCase2CamelCase` 结果为小驼峰，因此最后需要再 `Capitalize` 一下，这么做是为了避免在一个工具类型里实现两种不同的转换逻辑。如果希望直接转大驼峰，可以这么写：
>
> ```ts
> type KebabCase2CamelCase<S extends string> =
> S extends `${infer Head}${'-'}${infer Rest}`
>  ? `${Capitalize<Head>}${KebabCase2CamelCase<Capitalize<Rest>>}`
>  : Capitalize<S>;
>
> type ToEventHandler<TEvent extends string> = `on${KebabCase2CamelCase<TEvent>}`;
> ```
>
> 效果也是一样滴\~

现在就没问题了：

![](https://s3.bmp.ovh/imgs/2024/01/27/863911802c2267e1.png)

那要把这个联合类型转成一个属性就简单了，可以使用索引签名类型，即 TypeScript 内置的 Record 类型：

```ts
type VoidFunc = () => void;

type EventHandlerProperties = Record<EventHandlers, VoidFunc>;

interface VideoPlayerProps extends EventHandlerProperties {}

const VideoPlayer: React.FC<VideoPlayerProps> = ({ onPlay, onBlockVideo }) => {
  return <></>;
};
```

上面的例子看起来已经很不错了，但实际上还有一个严重的问题——它默认了所有事件都是一个 `VoidFunc` 类型，但通常来说会有一些事件拥有特殊的入参与出参，比如如果 `onClose` 的 EventHandler 返回了 false，就意味着当前关闭播放失败了。

这种情况下，我个人推荐引入一个新的类型结构来描述特定事件的处理器：

```ts
type EventHandlerType = {
  onUpdate: (input: { position: number }) => void;
  onClose: (input: { force: boolean }) => boolean;
};
```

然后改写下之前的 `EventHandlerProperties`，把它变成一个由两个部分组成的新类型：

* `EventHandlerType` 定义了的部分，这里的事件处理器类型需要使用 `EventHandlerType` 中定义的类型。
* `EventHandlerType` 未定义的部分，继续使用 `VoidFunc` 即可。

可以使用 Exclude 工具类型来完成这样的操作，把 `EventHandlerType` 定义的事件处理器从 `EventHandlers` 中去掉，整体的代码大概是酱婶儿的：

```ts
type EventHandlerType = {
  onUpdate: (input: { position: number }) => void;
  onClose: (input: { force: boolean }) => boolean;
};

type EventHandlerProperties = EventHandlerType &
  Record<Exclude<EventHandlers, keyof EventHandlerType>, VoidFunc>;

interface VideoPlayerProps extends EventHandlerProperties {}

const VideoPlayer: React.FC<VideoPlayerProps> = ({
  onUpdate,
  onBlockVideo,
}) => {
  onUpdate({ position: 7777 });

  return <></>;
};
```

除了作为 React 组件的属性类型，还有一个相当常见的场景：EventEmitter，最理想的效果是 EventEmitter 的每个事件监听注册都能为事件处理器提供对应的类型描述：

```ts
const event = new EventEmitter();

event.on('play', () => {});
event.on('update', ({ position }) => {});
```

要实现这种效果也很简单，首先，将事件名作为泛型参数的类型来源，然后用来匹配对应的 Handler：

```typescript
type GetHandlerType<TEvent extends Events> = ...?

class EventEmitter {
  public on<TEvent extends Events>(
    event: TEvent,
    handler: GetHandlerType<TEvent>
  ) {
    // ...
  }
```

整体思路其实和上面差不多，如果 `EventHandlerType` 定义了特殊的处理器类型，就使用其中的类型，否则使用朴素的 VoidFunc:

```typescript
type GetHandlerType<TEvent extends Events> =
  ToEventHandler<TEvent> extends keyof EventHandlerType
    ? EventHandlerType[ToEventHandler<TEvent>]
    : VoidFunc;
```

![](https://s3.bmp.ovh/imgs/2024/01/28/9e066d968e2d1f3a.png)

这里我们使用的是将「事件」转换为「事件处理器」后进行匹配的方式，如果你不嫌事多，其实也可以反过来，把 `keyof EventHandlerType` 都转换为「事件」的结构，给一个参考的实现：

```typescript
type CamelCase2KebabCase<
  S extends string,
  Start = true
> = S extends `${infer Head}${infer Rest}`
  ? `${Head extends Capitalize<Head>
      ? Start extends true
        ? Lowercase<Head>
        : `-${Lowercase<Head>}`
      : Head}${CamelCase2KebabCase<Rest, false>}`
  : S;

type CamelCase2KebabCase_Res_1 = CamelCase2KebabCase<'Update'>; // 'update'
type CamelCase2KebabCase_Res_2 = CamelCase2KebabCase<'BlockVideo'>; // 'block-video'
```

总结一下，这个例子里我们了解了一下更接地气一点的模板字符串类型编程，我可以打赌在你曾经写的项目里，肯定存在着可以用这种方式优化的类型定义。这也是我个人认为类型编程最有意义的场景之一：**只需要手动定义一次类型，后续的类型均使用类型编程得到**，这样一来，无论过程有多么复杂，后续你和你的接班人都不用再操心，只要改改手动定义的类型就好。

### 总结

在这两篇文章里，我们大致学习了两大类来自于社区工具的类型编程，它们分别关注「泛型的推导与转换」与「模板字符串类型的模式匹配」（严格来说所有类型编程都是在做泛型的推导与转换，这里只是便于概念区分强行分类）。

如果完整地学习下来，你会发现其实它们也没有想得那么复杂，其核心逻辑是可以在 <100 行的代码里实现的，当然，框架要处理的场景肯定复杂得多，因此其各种条件类型嵌套就显得繁复纷杂。

最后，在这几个示例的实现过程中，你可能发现了我一直在强调几个思考方式：

* 梳理泛型信息的传递链路，先想清楚泛型是否需要约束，从哪个参数来，到哪个结构去，中间需要如何转换，最终使用的效果是什么样的，想清楚了这些才能做到思维清晰。
* 别急着一步到位，我们的每一个例子都是从刀耕火种慢慢过渡到现代社会的，只要你不提交草稿，谁知道你背地里改了多少次实现呢。
* 对各种工具类型进行分类，比如 Prisma 实现中的集合工具类型，其它常见种类还有属性工具类型，结构工具类型，模式匹配工具类型等等等等...，这么做的好处就是你能根据实际的场景很快反应过来，哦这里该用榔头，哦这里该用剪刀...。

类型编程不可怕，可怕的是你把它想得太高深啦。我们下篇文章见:-)

### TypeScript 中的模板字符串类型

在 Hono 的例子中，我想你已经发现了模板字符串类型的妙用——它使得 TypeScript 可以从我们的字符串中来提取类型信息并用于类型检查。如果你此前并不了解模板字符串类型，这里我们做一个简要的介绍，大部分的内容同步自笔者此前在专栏发布的 [TypeScript 的另一面：类型编程](https://juejin.cn/post/7000360236372459527#heading-19)。

[TypeScript 4.1](https://devblogs.microsoft.com/typescript/announcing-typescript-4-1/) 中引入了模板字面量类型，使得我们可以使用`${}` 这一语法来构造字面量类型，如：

```typescript
type World = 'world';

// "hello world"
type Greeting = `hello ${World}`;
```

模板字面量类型具有自动分发的特性，即如果模板插槽中接收到的是一个联合类型，那么它会为联合类型的每一个成员进行一次调用，再将结果组合成一个联合类型，如：

```typescript
type SizeRecord<Size extends string> = `${Size}-Record`

// "Small-Record" | "Middle-Record" | "Huge-Record"
type UnionSizeRecord = SizeRecord<"Small" | "Middle" | "Huge">
```

如果存在多个插槽，各个联合类型将会被分别排列组合。

```typescript
type Struct = 'Record' | 'Array';

type SizeStruct<
  Size extends string,
  Struct extends string
> = `${Size}-${Struct}`;

//  "Small-Record" | "Middle-Record" | "Huge-Record"
// | "Small-Array" | "Middle-Array" | "Huge-Array"
type UnionSizeStruct = SizeStruct<Sizes, Struct>;
```

随之而来的还有四个新的工具类型：

```typescript
type Uppercase<S extends string> = intrinsic;

type Lowercase<S extends string> = intrinsic;

type Capitalize<S extends string> = intrinsic;

type Uncapitalize<S extends string> = intrinsic;
```

它们的作用就是字面意思，这里不做解释了。`intrinsic`代表了这些工具类型是由 TS 编译器内部实现的，其实也很好理解，我们无法通过类型编程来修改字面量的值；

TS 的实现代码：

```ts
function applyStringMapping(symbol: Symbol, str: string) {
  switch (intrinsicTypeKinds.get(symbol.escapedName as string)) {
    case IntrinsicTypeKind.Uppercase:
      return str.toUpperCase();
    case IntrinsicTypeKind.Lowercase:
      return str.toLowerCase();
    case IntrinsicTypeKind.Capitalize:
      return str.charAt(0).toUpperCase() + str.slice(1);
    case IntrinsicTypeKind.Uncapitalize:
      return str.charAt(0).toLowerCase() + str.slice(1);
  }
  return str;
}

```

你可能会想到，模板字面量如果想截取其中的一部分要怎么办？这里可没法调用 slice 方法。但我们还有 infer！使用 infer 占位后，便能够提取出字面量的一部分，如：

```typescript
type CutStr<Str extends string> = Str extends `${infer Part}budu` ? Part : never

// "lin"
type Tmp = CutStr<"linbudu">
```

再进一步，`[1,2,3]`这样的字符串，如果我们提供 `[${infer Member1}, ${infer Member2}, ${infer Member}]` 这样的插槽匹配，就可以实现神奇的提取字符串数组成员效果：

```typescript
type ExtractMember<Str extends string> = Str extends `[${infer Member1}, ${infer Member2}, ${infer Member3}]` ? [Member1, Member2, Member3] : unknown;

// ["1", "2", "3"]
type Tmp = ExtractMember<"[1, 2, 3]">
```

注意，这里的模板插槽被使用 `,` 分隔开了，如果多个带有 infer 的插槽紧挨在一起，那么前面的 infer 只会获得单个字符，最后一个 infer 会获得所有的剩余字符（如果有的话），比如我们把上面的例子改成这样：

```typescript
type ExtractMember<Str extends string> = Str extends `[${infer Member1}${infer Member2}${infer Member3}]` ? [Member1, Member2, Member3] : unknown;

// ["1", ",", " 2, 3"]
type Tmp = ExtractMember<"[1, 2, 3]">
```

这一特性使得我们可以使用多个相邻的 infer + 插槽，对最后一个 infer获得的值进行递归操作，如：

```typescript
type JoinArrayMember<T extends unknown[], D extends string> =
  T extends [] ? '' :
  T extends [any] ? `${T[0]}` :
  T extends [any, ...infer U] ? `${T[0]}${D}${JoinArrayMember<U, D>}` :
  string;

// ""
type Tmp1 = JoinArrayMember<[], '.'>;
// "1"
type Tmp3 = JoinArrayMember<[1], '.'>;
// "1.2.3.4"
type Tmp2 = JoinArrayMember<[1, 2, 3, 4], '.'>;
```

原理也很简单，每次将数组的第一个成员添加上`.`，在最后一个成员时不作操作，在最后一次匹配（`[]`）返回空字符串即可。

更花里胡哨一点，我们还可以在类型层面实现 Lodash 的 get 方法，即通过 `get({},"a.b.c")` 的形式快速获得嵌套属性，只需要结合 infer + 条件类型即可。

```typescript
type PropType<T, Path extends string> =
    string extends Path ? unknown :
    Path extends keyof T ? T[Path] :
    Path extends `${infer K}.${infer R}` ? K extends keyof T ? PropType<T[K], R> : unknown :
    unknown;

declare function getPropValue<T, P extends string>(obj: T, path: P): PropType<T, P>;
declare const s: string;

const obj = { a: { b: {c: 42, d: 'hello' }}};

getPropValue(obj, 'a');  // { b: {c: number, d: string } }
getPropValue(obj, 'a.b');  // {c: number, d: string }
getPropValue(obj, 'a.b.d');  // string
getPropValue(obj, 'a.b.x');  // unknown
getPropValue(obj, s);  // unknown
```
