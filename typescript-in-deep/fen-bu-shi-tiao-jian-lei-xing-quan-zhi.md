---
description: '2022-02-26'
---

# 分布式条件类型全知

> 本文是对在极客时间 与 [早早聊](https://www.zaozao.run/conf/c37) 的直播 中提到的 **分布式条件类型** 的进一步说明，但你不需要已经观看过相关直播，本文会包括前置知识部分。

**注意：本文的重点是分布式条件类型，对于前置的条件类型部分不会有特别深入的讲解，但其实也够了。**

本文的主要内容仍然主要来自于对之前的 [TypeScript的另一面：类型编程（2021重制版）](https://juejin.cn/post/7000360236372459527) 一文中，对分布式条件类型的讲解，也推荐仍处于学习阶段的同学完整的阅读这篇文章，我敢保证看完你的 TypeScript 水平就已经超越 79.8% 的使用者了。

### 条件类型

在本文中，我们只会介绍 extends 语句在常见情况下的成立情况，其他与条件类型相关的知识如原理、infer 关键字、基于条件类型的泛型约束等不做介绍（因为和本文主旨并冇关系）。

常见的 extends 语句可以分成这么几种成立情况：

* 字面量类型及其原始类型
* 基类与派生类的子类型关系
* 基于结构化类型系统的子类型关系
* 联合类型与其分支的子类型关系
* Top Type 与 Bottom Type
* 分布式条件类型

我们会一一介绍。

#### 字面量类型及其原始类型

字面量类型包括数字字面量、字符串字面量以及布尔字符串类型，它们是比原始类型更有意义的类型信息，如我们可以标注一个可选值确定的返回值：

> 一般认为模板字符串类型近似于字面量类型。

```typescript
type ResCode = 200 | 400 | 500;
```

我们可以标注一个确定的字符串集合：

```typescript
interface Foo {
  status: 'success' | 'failure'
}
```

也可以混用：

```typescript
type Mixed = true | 599 | 'Linbudu'
```

本质上，对于字面量类型，其可以理解为它是对原始类型的进一步收敛，是比原始类型更精确地类型，因此对于 extends 语句其必然成立。

```typescript
// true
type _T1 = 'linbudu' extends string ? true : false;
```

从另一种角度来说，我们也可以认为**字面量类型其实就是继承自原始类型**，如以下伪代码：

```typescript
class LinbuduLiteralType extends String {
  public value = 'Linbudu';
}
```

#### 子类型关系

对于基类和派生类的子类型关系，这一部分不做解释，直接上代码：

```typescript
class Base {
  name!: string;
}

class Derived extends Base {
  age!: number;
}

// true
type _T1 = Derived extends Base ? true : false;
```

还存在着一种类似的情况，即**结构化类型系统判断得到的子类型关系**：

> 关于结构化类型系统和标称类型系统的差异，咱们以后再说。

```typescript
type _T2 = { name: 'linbudu'; } extends Base
  ? true
  : false;

type _T3 = { name: 'linbudu'; age: 18; job: 'engineer' } extends Base
  ? true
  : false;
```

在这里我们手动定义的对象并没有真的 extends Base Class，但由于其内部的属性与 Base 类型一致（\_T3 还额外进行了扩展），结构化类型系统通过比较内部的属性与属性类型来判断类型的兼容性，因此这里 extends 同样成立。

还有一种更特殊的情况，即对于空对象`{}` 的比较：

```typescript
type _T4 = {} extends {}
  ? true
  : false;

type _T5 = { name: 'linbudu';  } extends {}
  ? true
  : false;
  
type _T6 = string extends {}
  ? true
  : false;
```

本质上类似于基类以及派生类，但空对象由于其内部无属性，任意一个对象（甚至是原始类型）都可以认为是它的子集。

#### 联合类型及其分支

对于联合类型的比较，其实就是比较前者的类型分支在后者中是否都存在，或者说**前者是否是后者的子集**：

```typescript
// true
type _T7 = 'a' extends 'a' | 'b' | 'c' ? true : false;

// true
type _T8 = 'a' | 'b' extends 'a' | 'b' | 'c' ? true : false;

// false
type _T9 = 'a' | 'b' | 'wuhu!' extends 'a' | 'b' | 'c' ? true : false;
```

在分布式条件类型一节中，我们会了解更多。

#### Top Type 与 Bottom Type

> 是的，这又是一个独立的知识点，我们还是以后... 所以你知道 TypeScript 类型系统的知识有多重要了吧。

在 TypeScript 我们说 any 与 unknown 是 **Top Type**，而 never 则是 **Bottom Type**。Top Type 意味着它们处于类型层级的顶端，即任意的类型都属于其子类型，对于 `OtherType extends any` 与 `OtherType extends unknown` 必定成立。而 Bottom Type 处于类型层级的底端，意味着无法再细分的类型，除了 never 自身，没有别的类型能够再赋值给它。这也就意味着，它属于任意类型的子类型，即 `never extends OtherType` 必定成立。

那么，一环一环的套起来，我们就可以构造出一条 extends 链，来直观的分析 TypeScript 的类型层级了：

```typescript
// 8，即所有 extends 均成立
type _Chain = never extends 'linbudu'
  ? 'linbudu' extends 'linbudu' | 'budulin'
    ? 'linbudu' extends string
      ? string extends {}
        ? {} extends Object
          ? Object extends any
            ? Object extends unknown
              ? any extends unknown
                ? unknown extends any
                  ? 8
                  : 7
                : 6
              : 5
            : 4
          : 3
        : 2
      : 1
    : 0
  : never;
```

### 分布式条件类型

分布式条件类型（Distributive Conditional Types）是 TypeScript 中条件类型的特殊功能之一，因此也被称为条件类型的分布式特性。

对于它其实没有什么特别晦涩难懂的弯弯绕绕，就是满足一定条件后必然会发生的事情罢了，就好像饿了要吃饭，困了要睡觉一样。所以没必要也不应该敬畏它，看我怎么把它扒光在你们面前。

来看一个例子，对于内置工具类型 Extract 的使用：

```typescript
type Extract<T, U> = T extends U ? T : never;
```

```typescript
interface IObject {
  a: string;
  b: number;
  c: boolean;
}

// 'a'|'b'
type _ExtractedKeys1 = Extract<keyof IObject, 'a'|'b'>;

// 'a'|'b'
type _ExtractedKeys2 = Extract<'a'|'b'|'c', 'a'|'b'>;

// never
type _ExtractedKeys3 = 'a'|'b'|'c' extends 'a'|'b' ? 'a'|'b'|'c' : never;
```

本质上，这三个类型别名执行的代码是一模一样的，但是为什么 3 和 1、2 表现得不一致（虽然这个 extends 不成立我们是理解的）？

研究下两种情况的区别，你会发现 1、2 是通过传入给泛型参数，再由泛型参数进行判断的，而 3 则是直接使用联合类型进行的判断。

记住第一个差异：**是否作为泛型参数**。

再看一个例子：

```typescript
type Naked<T> = T extends boolean ? "Y" : "N";
type Wrapped<T> = [T] extends [boolean] ? "Y" : "N";

// "N" | "Y"
type Result1 = Naked<number | boolean>;

// "N"
type Result2 = Wrapped<number | boolean>;
```

这里倒是都通过泛型参数进行判断了，但结果又不一样了，为什么第一个得到了联合类型？第二个倒是能理解，`[1, true]` 肯定和 `[true, true]` 不兼容嘛。

对于第一个，你可能已经发现这里作为结果的联合类型，恰好对应上了分别使用 number、boolean 判断结果，就好像于谦的父亲也姓于一样这么巧？那为什么第二个没有被这样子拆开比较？

记住第二个差异：**泛型参数在条件类型是否被数组包裹了**。

其实这里的两个差异就是分布式条件类型要发生的前提条件：

* 首先，你得是联合类型
* 其次，你的联合类型得是通过泛型参数的形式传入
* 最后，你的泛型参数在条件类型语句中需要是裸类型参数，即没有被 \[] 包裹

合起来，我们就得到了官方的解释：**对于属于裸类型参数的检查类型，条件类型会在实例化时期自动分发到联合类型上。**

> Conditional types in which the checked type is a naked type parameter are called distributive conditional types. Distributive conditional types are automatically distributed over union types during instantiation.

现在我们可以讲讲这里的分布式代表啥了，第一个例子中实际上是这么一个判断过程：

```typescript
// ('a' extends 'a'|'b') | ('b' extends 'a'|'b') | ('c' extends 'a'|'b')
// 'a'|'b'|never
// 'a'|'b'
type _ExtractedKeys1 = Extract<keyof IObject, 'a'|'b'>;
```

即联合类型的分支被单独的拿出来依次进行条件类型语句的判断。类似的，第二个例子在 Naked 工具类型也会将传入的联合类型参数进行分发判断，而 Wrapped 由于不满足裸类型参数的条件，导致分布式不成立，因此不会进行分发。

你可能会好奇，为啥要专门设计分布式条件类型这个东西？实际上，可以说它也是 TypeScript 类型系统的基石之一，大量的工具类型底层都依赖了它来进行联合类型的过滤，然后基于联合类型的过滤去做映射类型、Pick/Omit 这一类接口裁剪操作，准备开始实战环节！

### 分布式条件类型的应用

TypeScript 内置了几个基于分布式条件类型的工具类型：

```typescript
type Extract<T, U> = T extends U ? T : never;

type Exclude<T, U> = T extends U ? never : T;

type NonNullable<T> = T extends null | undefined ? never : T;
```

它们的作用分别是：

* Extract，提取 集合T 中也存在于 集合U 中的类型分支，即 T 与 U 的交集
* Exclude，提取 集合T 中不存在于 集合U 中的类型分支，即 T 相对于 U 的差集
* NonNullable，移除集合中的 null 与 undefined

它们的工作机理就是分布式条件类型了，看到了差集与交集，你的数学 DNA 是否动了？

当然，我们可以很容易的实现并集与补集：

```typescript
export type Concurrence<A, B> = A | B;

export type Complement<A, B extends A> = Exclude<A, B>;
```

勉为其难画个图好了，我可不是随便画图的人。

![并集、补集、差集、交集](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35864d6941664ac8acd76ad61eb08b07\~tplv-k3u1fbpfcp-zoom-1.image)

唯一需要注意的就是补集，补集实际上是特殊情况的差集，即 U 为 T 的子集，此时有 `T 相对于 U 的差集` + `U` = `T`。

这里我们实现了普通集合的情况，如果眉头一皱，你会发现，问题并不简单。如果我们面临的是对象的情况呢？这个时候如果我们取对象并集，那就没那么简单了，比如，如果一个键在两个对象中都存在，那么以哪个对象的键值类型为准？又比如，如果我们想实现合并对象时，只使用新对象的键值类型覆盖掉原对象的同键键值类型，但不想合并新的键过来？

这其实就是类型编程中，我称之为一刀两端藕断丝连再续前缘重归于好的操作，其实就是把一个接口，拆成多个部分，对某一部分做处理，然后再合并。这一思想在许多工具类型中都有体现，如：

```typescript
export type MarkPropsAsOptional<
  T extends Record<string, any>,
  K extends keyof T = keyof T
> = Omit<T, K> & Partial<Pick<T, K>>;
```

这个工具类型的作用是将接口中指定的一部分变为可选，而不像 Partial 那样全量的变为可选。它的思路就是把接口拆成 保持不变的 + 需要标记为可选的，然后对后一部分应用 Partial 类型，再合并即可。

回到对象的覆盖，其实思路是一样的。

* 合并 T 和 U 的所有键值对，以 U 的键值类型优先级更高
  * T 比 U 多的部分：T 相对于 U 的差集
  * U 比 T 多的部分：U 相对于 T 的差集
  * T 与 U 的交集，通过 `Extract<U, T>`，将 T 视为后入集合，实现以 U 为主集合，即 U 的类型优先级更高。

在开始前，我们还需要来实现对象交集、对象差集等几个辅助工具类型，以及辅助它们的对象键集合交集、对象键集合差集的辅助辅助工具类型（笑。为了方便理解，我们把 Extract 命名为 Intersection， Exclude 命名为 Difference。

```typescript
type PlainObjectType = Record<string, any>;

export type Intersection<A, B> = A extends B ? A : never;

export type Difference<A, B> = A extends B ? never : A;

export type ObjectKeysIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Intersection<keyof T, keyof U> & Intersection<keyof U, keyof T>;

export type ObjectKeysDifference<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Difference<keyof T, keyof U>;

export type ObjectIntersection<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Pick<T, ObjectKeysIntersection<T, U>>;

export type ObjectDifference<
  T extends PlainObjectType,
  U extends PlainObjectType
> = Pick<T, ObjectKeysDifference<T, U>>;
```

这样就可以实现以新对象类型优先级更高的 Merge 了：

```typescript
type Merge<
  T extends PlainObjectType,
  U extends PlainObjectType
  // T 比 U 多的部分，加上 T 与 U 交集的部分(类型不同则以 U 优先级更高，再加上 U 比 T 多的部分即可
> = ObjectDifference<T, U> & ObjectIntersection<U, T> & ObjectDifference<U, T>;
```

类似的，如果要保证原对象类型优先级更高，反转下交集即可：

```typescript
type Assign<
  T extends PlainObjectType,
  U extends PlainObjectType
  // T 比 U 多的部分，加上 T 与 U 交集的部分(类型不同则以 T 优先级更高，再加上 U 比 T 多的部分即可
> = ObjectDifference<T, U> & ObjectIntersection<T, U> & ObjectDifference<U, T>;
```

### 扩展

#### & 与 |

可能前面有的同学注意到了，我们的普通集合交集使用的是 | ，但对象的交集使用的是交叉类型 &：

```typescript
export type Concurrence<A, B> = A | B;

// 此 IntersectionTypes 非彼 Intersection
// 懒得写约束了，反正都得是对象类型就是了，西内！
export type IntersectionTypes<T, U, K> = T & U & K;
```

这是因为，只有对于对象类型，& 才表现为合并的情况，而对于原始类型以及联合类型，& 就真的表现为，交集。

```typescript
// 'a'
type _T1 = ('a' | 'b') & ('a' | 'd' | 'e' | 'f')

// never，因为 string 和 number 哪有交集啊
type _T1 = string & number;
```

#### 家庭作业

小明想要把两个对象 A、B 合并在一起，但不想要对象 B 中比对象 A 多的部分，而是只想把 B 中同键的键值类型合并过来，你能帮帮他吗？
