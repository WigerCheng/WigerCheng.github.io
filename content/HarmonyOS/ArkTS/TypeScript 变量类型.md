---
draft: true
tags:
  - HarmonyOS/ArkTS
---

# 变量声明

- `let` 
	- `let` 声明可以在不进行初始化的情况下完成
- `const`
	-  `const` 声明始终使用值进行初始化
	-  `const` 声明分配后，就无法再重新分配

```ts
let x: number;   //* Explicitly declares x as a number type
let y = 1;       //* Implicitly declares y as a number type
let z;           //* Declares z without initializing it
```
- 显示声明 `x` 是 number 类型。
- TypeScript 自动推断 `y` 的类型是 `number` 类型。
- TypeScript 自动推断 `z` 的类型是 `any` 类型。

# TypeScript类型
- ❗❗❗TypeScript中所有类型都是**[[#任何类型（any）]]** 的子类型。
- ![[typescript_type.png]]

# 基元类型（Primitive types）
- 基元类型是 `boolean`，`number`、`string`、`void`、`null` 和 `undefined` 类型以及用户定义的枚举或 `enum` 类型。
-  `void` 类型的存在纯粹是为了指示不存在值，例如存在于没有返回值的函数中。
-  `null` 和 `undefined` 类型是所有其他类型的子类型。无法显式引用 `null` 和 `undefined` 类型。 使用 `null` 和 `undefined` 字面量只能引用这些类型的值。

### 布尔类型(boolean)
- 布尔类型有两种字面量：`true`和`false`。
- 📕不要混淆作为[**布尔对象**](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Boolean)的真和假与布尔类型的原始值 true 和 false；布尔对象是原始布尔数据类型的一个包装器。
```ts
let flag: boolean;
let yes = true;
let no = false;
```

### 数字类型和大整数类型(number、bigint)
- TypeScript 中的所有数字都是浮点数或大整数
- 浮点数的类型为 `number`，而大整数的类型为 [`bigint`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/BigInt)
- TypeScript 支持十六进制、十进制、八进制和二进制字面量
```ts
let x: number;
let y = 0;
let z: number = 123.456;
let big: bigint = 100n;//(整数字面量后面加n的方式定义一个BigInt)
```
> **整数字面量**
> 十进制整数字面量由一串数字序列组成，且没有前缀 0。
> 八进制的整数以 0（或 0O、0o）开头，只能包括数字 0-7。
> 十六进制整数以 0x（或 0X）开头，可以包含数字（0-9）和字母 a~f 或 A~F。
> 二进制整数以 0b（或 0B）开头，只能包含数字 0 和 1。

### 字符串类型(string)
- 使用双引号 (`"`) 或单引号 (`'`) 将字符串数据括起来。
- 在 TypeScript 中，还可以使用[模板字符串](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals)，该模板字符串可以跨越多行并具有嵌入式表达式。 这些字符串由反撇号/反引号 （`` ` ``）字符括起，并且嵌入式表达式的形式为 `${ expr }`。
```ts
let s: string;
let empty = "";
let abc = 'abc';

let firstName: string = "Mateo";
let sentence: string = `My name is ${firstName}.
    I am new to TypeScript.`;
console.log(sentence);

//My name is Mateo.
//    I am new to TypeScript.
```

### 枚举(enum)
- 枚举提供了一种处理相关常量集的简单方法。
- `enum` 是一组值的符号名。 枚举被视为数据类型，你可以使用它们来创建用于变量和属性的常量集。
#### Numeric enums
```ts
enum Direction {
	Up,
	Down,
	Left,
	Right,
}
```
- 默认情况下，枚举第一位的值是0，每一位后面的枚举值自增1；同时也可以显式指定枚举值，*如将Down=12345，则Left的值为12346，Right的值为12347*。
- ❗❗❗值可以不唯一，即Up,Down,Left,Right 均可以指定相同的值。
```ts
enum Direction {
     Up = 10086,
     Down,
     Left = 12580,
     Right,
}

let DirectionUp : Direction = Direction.Up;
let DirectionDown : Direction = Direction.Down;
let DirectionLeft : Direction = Direction.Left;
let DirectionRight : Direction = Direction.Right;

console.log(DirectionUp);//10086
console.log(DirectionDown);//10087
console.log(DirectionLeft);//12580
console.log(DirectionRight);//12581
console.log(Direction[DirectionUp]);//"Up"
```
#### String enums
```ts
enum Direction {
	Up = "UP",
	Down = "DOWN",
	Left = "LEFT",
	Right = "RIGHT",
}

console.log(DirectionUp);//"UP"
console.log(DirectionDown);//"DOWN"
console.log(DirectionLeft);//"LEFT"
console.log(DirectionRight);//RIGHT"
```
- 方便debug和可读性。

# 任何类型（any）
- `any` 类型是可以无限制地表示任何 JavaScript 值的一种类型。当你期望某个值来自第三方库或值为动态的用户输入时，此类型很有用，**因为 `any` 类型将允许重新分配不同类型的值**。
-  `any` 类型选择不进行类型检查，并且不会强制你在调用、构造或访问这些值的属性之前进行任何检查，不会出现生成编译的错误，但可能会出现运行错误。
```ts
let randomValue: any = 10;
randomValue = 'Mateo';   // OK
randomValue = true;      // OK	

console.log(randomValue.name);  // Logs "undefined" to the console
randomValue();                  // Returns "randomValue is not a function" error
randomValue.toUpperCase();      // Returns "randomValue is not a function" error
```
- ❗❗❗`any` 的所有便利都以失去类型安全性为代价。 类型安全是使用 TypeScript 的主要动机之一。 如果不需要，应避免使用 `any`。

## unknown 类型
- `unknown` 类型与 `any` 类型的相似之处在于，可以将任何值赋予类型 `unknown`。 但无法访问 `unknown` 类型的任何属性，也不能调用或构造它们。
```ts
let randomValue: unknown = 10;
randomValue = true;
randomValue = 'Mateo';

console.log(randomValue.name);  // Error: Object is of type unknown
randomValue();                  // Error: Object is of type unknown
randomValue.toUpperCase();      // Error: Object is of type unknown
```
- ❗❗❗`any` 和 `unknown` 之间的核心区别在于你无法与 `unknown` 类型的变量进行交互；这样做会产生“编译器”错误。 `any` 将绕过所有编译时检查，并且在运行时评估对象；如果该方法或属性存在，它将表现出预期的效果。
# 对象类型和类型参数（Object types、Type parameters）
- 对象类型是所有类、接口、[[#数组]]和字面量类型（不是基元类型的任何类型）。
- 类和接口类型将通过类和接口声明引入，并通过在其声明中为其指定的名称进行引用。 类和接口类型可以是具有一个或多个类型参数的通用类型。

# 类型断言（`as`、`<>`）
- 如果需要将变量视为其他数据类型，则可以使用类型断言。
-  类型断言告诉 TypeScript 你在调用该语句之前已执行了所需的任何特殊检查。 它告诉编译器“相信我，我知道我在做什么”。
- 类型断言就像其他语言中的类型转换一样，**但是它不执行数据的特殊检查或重组。它对运行时没有影响，仅由编译器使用。**
- 类型断言两种形式：
	1.  `as` 语法（首选）：`(randomValue as string).toUpperCase();`
	2. “尖括号”语法：`(<string>randomValue).toUpperCase();`
```ts
///在使用类型断言调用 `toUpperCase` 方法之前，执行必要的检查以确定 `randomValue` 是 `string`
// TypeScript 假定你已进行必要的检查。 类型断言指出 `randomValue` 应该被视为 `string`，然后可以应用 `toUpperCase` 方法。
let randomValue: unknown = 10;

randomValue = true;
randomValue = 'Mateo';

if (typeof randomValue === "string") {
    console.log((randomValue as string).toUpperCase());    //* Returns MATEO to the console.
} else {
    console.log("Error - A string was expected here.");    //* Returns an error message.
}
```

# 类型保护（`typeof`）
- 除了 [Object](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Data_structures#object) 以外，所有类型都定义了表示在语言最低层面的[不可变](https://developer.mozilla.org/zh-CN/docs/Glossary/Immutable)值。[我们将这些值称为原始值](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Data_structures#%E5%8E%9F%E5%A7%8B%E5%80%BC)。
- 除了 [`null`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/null)，所有原始类型都可以使用 [`typeof`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/typeof) 运算符进行测试。`typeof null` 返回 `"object"`，**因此必须使用 `=== null` 来测试 `null`**。
- 在 `if` 块中使用 [`typeof`](https://www.typescriptlang.org/docs/handbook/2/typeof-types.html) 在运行时检查表达式的类型。 此条件测试称为“类型保护”。
```js
// 数值
typeof 37 === "number";
typeof 3.14 === "number";
typeof 42 === "number";
typeof Math.LN2 === "number";
typeof Infinity === "number";
typeof NaN === "number"; // 尽管它是 "Not-A-Number" (非数值) 的缩写
typeof Number(1) === "number"; // Number 会尝试把参数解析成数值
typeof Number("shoe") === "number"; // 包括不能将类型强制转换为数字的值

typeof 42n === "bigint";

// 字符串
typeof "" === "string";
typeof "bla" === "string";
typeof `template literal` === "string";
typeof "1" === "string"; // 注意内容为数字的字符串仍是字符串
typeof typeof 1 === "string"; // typeof 总是返回一个字符串
typeof String(1) === "string"; // String 将任意值转换为字符串，比 toString 更安全

// 布尔值
typeof true === "boolean";
typeof false === "boolean";
typeof Boolean(1) === "boolean"; // Boolean() 会基于参数是真值还是虚值进行转换
typeof !!1 === "boolean"; // 两次调用 !（逻辑非）运算符相当于 Boolean()

// Symbols
typeof Symbol() === "symbol";
typeof Symbol("foo") === "symbol";
typeof Symbol.iterator === "symbol";

// Undefined
typeof undefined === "undefined";
typeof declaredButUndefinedVariable === "undefined";
typeof undeclaredVariable === "undefined";

// 对象
typeof { a: 1 } === "object";

// 使用 Array.isArray 或者 Object.prototype.toString.call
// 区分数组和普通对象
typeof [1, 2, 4] === "object";

typeof new Date() === "object";
typeof /regex/ === "object";

// 下面的例子令人迷惑，非常危险，没有用处。避免使用它们。
typeof new Boolean(true) === "object";
typeof new Number(1) === "object";
typeof new String("abc") === "object";

// 函数
typeof function () {} === "function";
typeof class C {} === "function";
typeof Math.sin === "function";

//!!!!!!`typeof null` 返回 `"object"`
typeof null === "object";
//!!!!!!
```

### 常见typeof返回值
| 类型                                                                                                                                                                                                | 结果            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| [Undefined](https://developer.mozilla.org/zh-CN/docs/Glossary/Undefined)                                                                                                                          | `"undefined"` |
| [Null](https://developer.mozilla.org/zh-CN/docs/Glossary/Null)                                                                                                                                    | `"object"`    |
| [Boolean](https://developer.mozilla.org/zh-CN/docs/Glossary/Boolean)                                                                                                                              | `"boolean"`   |
| [Number](https://developer.mozilla.org/zh-CN/docs/Glossary/Number)                                                                                                                                | `"number"`    |
| [BigInt](https://developer.mozilla.org/zh-CN/docs/Glossary/BigInt)                                                                                                                                | `"bigint"`    |
| [String](https://developer.mozilla.org/zh-CN/docs/Glossary/String)                                                                                                                                | `"string"`    |
| [Symbol](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol)                                                                                                 | `"symbol"`    |
| [Function](https://developer.mozilla.org/zh-CN/docs/Glossary/Function)（在 ECMA-262 中实现 [[Call]]；[classes](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/class)也是函数) | `"function"`  |
| 其他任何对象                                                                                                                                                                                            | `"object"`    |
### 常见类型判断

| 类型          | Predicate                          |
| ----------- | ---------------------------------- |
| `string`    | `typeof s === "string"`            |
| `number`    | `typeof n === "number"`            |
| `boolean`   | `typeof b === "boolean"`           |
| `undefined` | `typeof undefined === "undefined"` |
| `function`  | `typeof f === "function"`          |
| `array`     | `Array.isArray(a)`                 |


# [联合类型与交叉类型](https://www.typescriptlang.org/docs/handbook/unions-and-intersections.html)
## 联合类型（`|`）
- 联合类型描述的值可以是几种类型之一。*当值不受控制时（例如，来自库、API 或用户输入的值），这种灵活性将很有帮助。*
- **联合类型将赋值限制为联合中的指定类型，而 `any` 类型没有限制。**
- **联合类型使用竖线 (`|`) 分隔每种类型。**
```ts
let multiType: number | boolean;
multiType = 20;         //* Valid
multiType = true;       //* Valid
multiType = "twenty";   //* Invalid
```

```ts
/// `add` 函数可接受两个值，它们可以是 `number` 或 `string`。 如果两个值都是数字类型，则将它们相加。 如果两者都是字符串类型，则将它们连接起来。 否则，将引发错误。
function add(x: number | string, y: number | string) {
    if (typeof x === 'number' && typeof y === 'number') {
        return x + y;
    }
    if (typeof x === 'string' && typeof y === 'string') {
        return x.concat(y);
    }
    throw new Error('Parameters must be numbers or strings');
}
console.log(add('one', 'two'));  //* Returns "onetwo"
console.log(add(1, 2));          //* Returns 3
console.log(add('one', 2));      //* Returns error
```
## 交叉类型（`&`）
- 交叉类型组合两个或多个类型以**创建**具有现有类型的所有属性的新类型。 使用交叉可以将现有类型加在一起，以获得**具有所需的所有功能的单个类型**。
- 交叉类型常用于接口interface。
```ts
///定义了两个接口 `Employee` 和 `Manager`，然后创建了一个称为 `ManagementEmployee` 的新交叉类型，该交叉类型将两个接口中的属性组合在一起
interface Employee {
  employeeID: number;
  age: number;
}
interface Manager {
  stockPlan: boolean;
}
type ManagementEmployee = Employee & Manager;
let newManager: ManagementEmployee = {
    employeeID: 12345,
    age: 34,
    stockPlan: true
};
```

# [字面量类型](https://www.typescriptlang.org/docs/handbook/literal-types.html)
- TypeScript 中提供了三组字面量类型：`string`、`number` 和 `boolean`。
- 通过使用字面量类型，你可以指定字符串，数字或布尔值**必须具有的确切值** *（例如，“是”、“否”或“或许”）*。
- 字面量类型以对象、数组、函数或构造函数类型字面量的形式编写，用于将其他类型组合为新类型。
```ts
type testResult = "pass" | "fail" | "incomplete";
let myResult: testResult;
myResult = "incomplete";    //* Valid
myResult = "pass";          //* Valid
myResult = "failure";       //* Invalid

type dice = 1 | 2 | 3 | 4 | 5 | 6;
let diceRoll: dice;
diceRoll = 1;    //* Valid
diceRoll = 2;    //* Valid
diceRoll = 7;    //* Invalid

interface ValidationSuccess {
	isValid: true;
	reason: null;
}

interface ValidationFailure {
	isValid: false;
	reason: string;
}

type ValidationResult = ValidationSuccess | ValidationFailure;
```

# 数组
```ts
let list: number[] = [1, 2, 3];//使用元素类型后跟方括号 (`[ ]`) 来表示该元素类型的数组
let list: Array<number> = [1, 2, 3];//通过语法 `Array<type>` 使用泛型 `Array` 类型
```

# 元组
- 元组类型允许表示一个已知元素数量和类型的数组，各元素的类型不必相同。
- `let person: [string, number] = ['Marcia', 35];` 一个包含 `string` 和 `number` 的元组。
	- ❌`let person: [string, number] = ['Marcia', 35, true];`元组中array的元素是固定的。
	- ❌`let person: [string, number] = [35, 'Marcia'];`元组中数组项值的顺序必须与类型的顺序匹配。
- **元组通过下标读取值；也可通过解构赋值读取值。**
```ts
type StringNumberPair = [string, number];

function doSomething(pair: [string, number]) {
    const a: string = pair[0];
    const b: number = pair[1];
    console.log(`a:${a} b:${b}`);
    const [inputString, hash] = pair;
    console.log(`inputString:${inputString} hash:${hash}`);
}
doSomething(["hello", 42]);
//a:hello b:42
//inputString:hello hash:42
```