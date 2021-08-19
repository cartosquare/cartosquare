---
title: Learn Typescript Part 1
published: true
---

:computer::computer:

TypeScript = JavaScript with static-type checking.

TypeScript相比于Javascript的优势在于：

* 避免类型错误。比如，把一个变量当成函数来调用:

```typescript
const message = "hello!";

message();
```
* 避免不存在对象/属性/函数的错误。比如，调用一个对象的不存在的属性:

```typescript
const user = {
	name: "Daniel",
  age: 26,
};

user.location;
```

或者把函数名称拼写错误：

```typescript
const announcement = "Hello World!";

// Meant to be toLocaleLowerCase()
announcement.toLocalLowerCase();
```

* 避免函数未调用的错误：

```typescript
function flipCoin() {
  // Meant to be Math.random()
  return Math.random < 0.5;
}
```
* 避免基本逻辑错误：

```typescript
const value = Math.random() < 0.5 ? "a" : "b";
if (value != "a") {
  // ...
} else if (value == "b") {
  // 这个条件内的代码永远不会被执行
}
```
* 可以在编辑器里提供类型错误信息（不需要编译），提供代码补全功能，快速修复错误功能

# `tsc`，TypeScript的编译器

## 安装

```
npm install -g typescript
```

## 编译

```
tsc source.ts
```

把ts文件编译为js文件。

## 常用编译选项

* --noEmitOnError

出现错误不更新目标文件

* --target

用于制定输出的ES规范，默认输出ES3规范，比如 `--target es2015` 指定输出 ECMAScript 2015规范的代码。

* --strict

要求编译器检测类型错误、null/undefined错误，以及隐式把类型推断为 `any` 的错误。

* --noImpliciAny

要求编译器检测隐式把类型推断为 `any` 的错误

* --strictNullChecks 

要求编译器检测null/undefined错误

# 数据类型

## 基本数据类型

* string

* number

* boolean

## 数组

类型后面跟 `[]`，比如，数值数组：`number[]`，字符串数组：`string[]`

## any

表示任何类型，尽量避免使用！

## 给变量指定类型

变量名称后面加冒号，跟上类型：
```typescript
let myName: string = "Cartosquare";
```

函数参数的类型和返回值也一样：

```typescript
function greet(name: string): string {
  return "Hello" + name.toUpperCase() + "!!";
}
```

## 对象类型

变量后面冒号加对象定义：

```typescript
// The parameter's type annotation is an object type
function printCoord(pt: { x: number; y: number }) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
printCoord({ x: 3, y: 7 });

function printName(obj: { first: string; last?: string }) {
  // ...
}
// Both OK
printName({ first: "Bob" });
printName({ first: "Alice", last: "Alisson" });
```

## Union 类型

变量后面冒号加 `｜` 操作符串联的类型：

```typescript
function printId(id: number | string) {
  console.log("Your ID is: " + id);
}
// OK
printId(101);
// OK
printId("202");
// Error
printId({ myID: 22342 });
```

## 类型别名

如果对象类型不止一处使用，可以使用类型别名：

```typescript
type POint = {
  x: number,
  y: number,
};

// Exactly the same as the earlier example
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });

type ID = number | string;
```

## 接口

接口和类型别名的作用十分相似，区别在于接口可以添加新的属性：

```typescript
interface Point {
  x: number,
  y: number,
};

function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });
```

## 类型断言

使用 `as` 操作符断定变量的类型：

```typescript
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```

## 常量类型

常量也是一种类型！:thinking:

```typescript
// Because `constantString` can only represent 1 possible string, it
// has a literal type representation
// It equals: let constantString: "Hello World" = "Hello World"
const constantString = "Hello World";
```

常量类型通常在 `Union` 中使用：

```typescript
function printText(s: string, alignment: "left" | "right" | "center") {
  // ...
}
printText("Hello, world", "left");
// function printText(s: string, alignment: "left" | "right" | "center") {
  // ...
}
printText("Hello, world", "left");
printText("G'day, mate", "centre");
// Argument of type '"centre"' is not assignable to parameter of type '"left" | "right" | "center"'.
printText("G'day, mate", "centre");
```

数值类型的常量类型使用方法也类似：

```typescript
function compare(a: string, b: string): -1 | 0 | 1 {
  return a === b ? 0 : a > b ? 1 : -1;
}
```

当然，常量类型也可以和非常量类型一起使用：

```typescript
interface Options {
  width: number;
}

function configure(x: Options | "auto") {
  // ...
}
configure({width: 100});
configure("auto");
// Error
configure("automatic");
```

## 常量推断

申明的常量对象的含义是类型不变，对象内的属性的值是可以改变的：

```typescript
const obj = { counter: 0 };
if (someCondition) {
  obj.counter = 1;
}
```

这里 `const` 的含义是值：`counter` 的类型是 `number`，而不是**常量变量0**。

同理，对于`string`也是一样：

```typescript
const req = { url: "https://some.com", method: "GET" };
```
这里 `method` 的类型是 `string`，而不是 `GET` 常量。

如果想要让属性也推断为常量，可以用 `as` 操作符，或者 `as const` 后缀：

```typescript
const req = { url: "https://example.com", method: "GET" as "GET" };

// or
onst req = { url: "https://example.com", method: "GET" } as const;
```

## `null` 和 `undefined` 类型

建议开启 `--strictNullChecks` 选项，并在代码里处理 `null` 的情形：

```typescript
function doSomething(x: string | null) {
  if (x === null) {
    // do nothing
  } else {
    console.log("Hello, " + x.toUpperCase());
  }
}
```

## 非空断言（`!`)

非空断言可以移除 `null` 和 `undefined` 错误，但是请确保值不可能是空的，否则，运行时还是会出错。

```typescript
function liveDangerously(x?: number | null) {
  // No error
  console.log(x!.toFixed());
}

```

## 不常用的基础类型

### bigint

ES2020之后才有的特性，用来表示特别大的数：

```typescript
// Creating a bigint via the BigInt function
const oneHundred: bigint = BigInt(100);
 
// Creating a BigInt via the literal syntax
const anotherHundred: bigint = 100n;
```

### symbol

`symbol` 可以用来创建全局唯一的引用：

```typescript
const firstName = Symbol("name");
const secondName = Symbol("name");
 
if (firstName === secondName) {
  // This condition will always return 'false' since the types 'typeof firstName' and 'typeof secondName' have no overlap.
  // Can't ever happen
}
```

# 参考资料

[JavaScript指南](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide)
[TypeScript HandBook](https://www.typescriptlang.org/docs/handbook/intro.html)