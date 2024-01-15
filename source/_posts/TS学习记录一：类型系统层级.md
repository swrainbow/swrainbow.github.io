---
title: TS学习记录一：类型系统层级
date: 2024-01-11 18:49:45
tags: [TS]
categories: [TS]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240115133807.png
---
# 基础
### type 与interface

- type 类型别名 更适合用来描述函数的签名、一组联合类型、一个工具类型
- interface 适合用来描述对象 类的结构
当然很多interface也可以用type直接替换，代码中如果统一使用也没什么问题

### object、Object、{}
原型链的顶端是 Object 以及 Function，这也就意味着所有的原始类型与对象类型最终都指向 Object和 Object 类似的还有 Boolean、Number、String、Symbol，这几个装箱类型（Boxed Types） 同样包含了一些超出预期的类型。以 String 为例，它同样包括 undefined、null、void，以及代表的 拆箱类型（Unboxed Types） string，但并不包括其他装箱类型对应的拆箱类
```typescript
const tmp9: String = undefined;
const tmp10: String = null;
const tmp11: String = void 0;
const tmp12: String = 'linbudu';

// 以下不成立，因为不是字符串类型的拆箱类型
const tmp13: String = 599; // X
const tmp14: String = { name: 'linbudu' }; // X
const tmp15: String = () => {}; // X
const tmp16: String = []; // X
```
在任何情况下，你都不应该使用这些装箱类型。
object 的引入就是为了解决对 Object 类型的错误使用，它代表所有非原始类型的类型，即数组、对象与函数类型这些：
```typescript
const tmp17: object = undefined;
const tmp18: object = null;
const tmp19: object = void 0;

const tmp20: object = 'linbudu';  // X 不成立，值为原始类型
const tmp21: object = 599; // X 不成立，值为原始类型

const tmp22: object = { name: 'linbudu' };
const tmp23: object = () => {};
const tmp24: object = [];
```
### 枚举和对象
枚举和对象的重要差异在于，对象是单向映射的，我们只能从键映射到键值。而枚举是双向映射的，即你可以从枚举成员映射到枚举值，也可以从枚举值映射到枚举成员：
> ```typescript
> enum Items {
>   Foo,
>   Bar,
>   Baz
> }
> 
> const fooValue = Items.Foo; // 0
> const fooKey = Items[0]; // "Foo"
> ```

> ```typescript
> "use strict";
> var Items;
> (function (Items) {
>     Items[Items["Foo"] = 0] = "Foo";
>     Items[Items["Bar"] = 1] = "Bar";
>     Items[Items["Baz"] = 2] = "Baz";
> })(Items || (Items = {}));
> ```
但需要注意的是，仅有值为数字的枚举成员才能够进行这样的双向枚举，字符串枚举成员仍然只会进行单次映射。
### Any & Unknown & Never
any 类型的主要意义，其实就是为了表示一个无拘无束的“任意类型”，它能兼容所有类型，也能够被所有类型兼容

> any 的本质是类型系统中的顶级类型，即 Top Type，这是许多类型语言中的重要概念，我们会在类型层级部分讲解。

any 类型的万能性也导致我们经常滥用它，比如类型不兼容了就 any 一下，类型不想写了也 any 一下，不确定可能会是啥类型还是 any 一下。此时的 TypeScript 就变成了令人诟病的 AnyScript。为了避免这一情况，我们要记住以下使用小 tips ：

- 如果是类型不兼容报错导致你使用 any，考虑用类型断言替代，我们下面就会开始介绍类型断言的作用。
- 如果是类型太复杂导致你不想全部声明而使用 any，考虑将这一处的类型去断言为你需要的最简类型。如你需要调用 `foo.bar.baz()`，就可以先将 foo 断言为一个具有 bar 方法的类型。
- 如果你是想表达一个未知类型，更合理的方式是使用 unknown。


unknown 类型和 any 类型有些类似，一个 unknown 类型的变量可以再次赋值为任意其它类型，但只能赋值给 any 与 unknown 类型的变量：
```typescript
let unknownVar: unknown = "linbudu";

unknownVar = false;
unknownVar = "linbudu";
unknownVar = {
  site: "juejin"
};

unknownVar = () => { }

const val1: string = unknownVar; // Error
const val2: number = unknownVar; // Error
const val3: () => {} = unknownVar; // Error
const val4: {} = unknownVar; // Error

const val5: any = unknownVar;
const val6: unknown = unknownVar;
```
简单地说，any 放弃了所有的类型检查，而 unknown 并没有
### 虚无的 never 类型
在编程语言的类型系统中，never 类型被称为 Bottom Type，是整个类型系统层级中最底层的类型。和 null、undefined 一样，它是所有类型的子类型，但只有 never 类型的变量能够赋值给另一个 never 类型变量。

### 索引类型
```ts
interface AllStringTypes {
  [key: string]: string;
}

type AllStringTypes = {
  [key: string]: string;
}
```

### 约束
```typescript
type ResStatus<ResCode extends number = 10000> = ResCode extends 10000 | 10001 | 10002
  ? 'success'
  : 'failure';

type Res4 = ResStatus; // "success"
```
分页结果的抽离
```typescript
interface IPaginationRes<TItem = unknown> {
  data: TItem[];
  page: number;
  totalCount: number;
  hasNextPage: boolean;
}

function fetchUserProfileList(): Promise<IRes<IPaginationRes<IUserProfileRes>>> {}
```
### 在 TypeScript 中模拟标称类型系统
```typescript
type USD = number;
type CNY = number;

const CNYCount: CNY = 200;
const USDCount: USD = 200;

function addCNY(source: CNY, input: CNY) {
  return source + input;
}

addCNY(CNYCount, USDCount)
```
再看一遍这句话：**类型的重要意义之一是限制了数据的可用操作与实际意义**。这往往是通过类型附带的**额外信息**来实现的（类似于元数据），要在 TypeScript 中实现，其实我们也只需要为类型额外附加元数据即可，比如 CNY 与 USD，我们分别附加上它们的单位信息即可，但同时又需要保留原本的信息（即原本的 number 类型）。

我们可以通过交叉类型的方式来实现信息的附加：

```typescript
export declare class TagProtector<T extends string> {
  protected __tag__: T;
}

export type Nominal<T, U extends string> = T & TagProtector<U>;
```

```typescript
export type CNY = Nominal<number, 'CNY'>;

export type USD = Nominal<number, 'USD'>;

const CNYCount = 100 as CNY;

const USDCount = 100 as USD;

function addCNY(source: CNY, input: CNY) {
  return (source + input) as CNY;
}

addCNY(CNYCount, CNYCount);

// 报错了！
addCNY(CNYCount, USDCount);
```
### 类型层级
型层级实际上指的是，TypeScript 中所有类型的兼容关系，从最上面一层的 any 类型，到最底层的 never 类型

#### 判断类型兼容性的方式
```typescript
declare let source: string;

declare let anyType: any;
declare let neverType: never;

anyType = source;

// 不能将类型“string”分配给类型“never”。
neverType = source;
```

对于变量 a = 变量 b，如果成立，意味着 <变量 b 的类型> extends <变量 a 的类型> 成立，即 b 类型是 a 类型的子类型，在这里即是 string extends never ，这明显是不成立的。

为什么需要 Top Type 与 Bottom Type ？ 在实际开发中，我们不可能确保对所有地方的类型都进行精确的描述，因此就需要 Top Type 来表示一个包含任意类型的类型。而在类型编程中，如果对两个不存在交集的类型强行进行交集运算，也需要一个类型表示这个不存在的类型。这就是 Top Type 与 Bottom Type 的存在意义。