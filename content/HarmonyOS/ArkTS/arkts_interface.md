---
title: ArkTS接口
draft: true
---

# 什么是接口
- 可以使用接口来描述对象、命名和参数化对象的类型，以及将现有的命名对象类型组成新的对象类型。
- 接口不会初始化或实现在其中声明的属性，其唯一作用是描述类型。它定义了**代码协定**所需的内容，而实现接口的变量、函数或类则通过提供所需的实现详细信息来满足协定。
```ts
//定义 `Employee` 对象的两个属性和一个方法的简单接口
interface Employee {
    firstName: string;
    lastName: string;
    fullName(): string;
}

//实现接口
let employee: Employee = {
    firstName : "Emil",
    lastName: "Andersson",
    fullName(): string {
        return this.firstName + " " + this.lastName;
    }
}
```
- 由于 TypeScript 具有结构化类型系统，因此可以认为一个具有特定成员集的接口类型与另一个具有相同成员集的接口类型或对象类型相同，并且可以用后者替换前者。即*如果一个接口和一个类实现相同的结构，则它们可以互换使用。*
- 📕*TypeScript 编码准则建议接口不应以字母 `I` 开头。*
- 接口可以[[#扩展接口|相互扩展]]。 这使你能够将一个接口的成员复制到另一个接口，从而在将接口分离为可重用组件方面提供了更大的灵活性。

# 接口的属性
- ❗ 接口属性类型属性可以为**必需、可选或只读**属性。

| 属性 类型 | 说明                                                              | 示例                            |
| ----- | --------------------------------------------------------------- | ----------------------------- |
| 必须    | 除非另行指定，否则所有属性都是必需的。                                             | `firstName: string;`          |
| 可选    | 在属性名称的末尾添加问号 (`?`)。 **对于不是必需的属性，请使用此属性。** 这可以防止类型系统在省略该属性时引发错误。 | `firstName?: string;`         |
| 只读    | 在属性名称的前面添加 readonly 关键字。 **对于只应在首次创建对象时修改的属性，请使用此属性。**          | `readonly firstName: string;` |

# 扩展接口
- 当使用一个或多个接口扩展接口时，将适用以下规则：
	- 必须从所有接口实现所有必需的属性。
	- 如果属性具有完全相同的名称和类型，则两个接口可以具有相同的属性。
	- 如果两个接口具有名称相同但类型不同的属性，则必须声明一个新属性，以使生成的属性是这两个接口的子类型。

```ts
interface IceCream {
    flavor: string,
    scoops: number;
    instructions?: string;
}
  
interface Sundae extends IceCream {
    sauce: 'chocolate' | 'caramel' | 'strawberry',
    nuts?: boolean,
    whippedCream?: boolean,
    instructions?: string;
}
  
let myIceCream: IceCream = {
    flavor: 'vanillna',
    scoops: 2
} 

let mySundae: Sundae = {
    flavor: 'vanilla',
    scoops: 2,
    sauce: 'caramel',
    nuts: true,
    instructions: "Vanilla Sundae"
} 

console.log(myIceCream);//{ flavor: 'vanillna', scoops: 2 }
console.log(mySundae);
// {
//     flavor: 'vanilla',
//     scoops: 2,
//     sauce: 'caramel',
//     nuts: true,
//     instructions: 'Vanilla Sundae'
//   }

function tooManyScoops(dessert: Sundae) {
    if (dessert.scoops >= 4) {
        return dessert.scoops + " is too many scoops!";
    } else {
        return "Your order will be ready soon!";
    }
}

console.log(tooManyScoops({ flavor: 'vanillna', scoops: 5, sauce: 'caramel' }))
//5 is too many scoops!
```

## TODO:可索引类型