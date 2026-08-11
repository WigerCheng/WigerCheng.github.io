---
title: "@Prop_父子单向同步"
tags:
  - "#StateV1"
  - HarmonyOS/ArkUI
---

## 原理

`@Prop` 装饰的变量用于**父子组件之间的单向状态同步**。当父组件绑定给 `@Prop` 的数据源发生改变时，相关变化会自动同步至子组件的 `@Prop` 变量，并**触发该子组件直接绑定的 UI 渲染更新**。

`@Prop` 装饰的变量允许在子组件内进行**本地修改**，但本地修改的变化**不会同步回父组件**；并且当父组件中的数据源再次发生更新时，子组件中的本地修改会被父组件传递的新值**覆盖**。

## 使用要求

1. `@Prop` 修饰的是组件内部状态属性，是私有的，只能通过组件内部访问。
2. `@Prop` 变量允许在本地赋初始值，也可由父组件传入初始化。若声明时未包含本地初始值，则**必须由父组件在创建子组件时传入初始值**；若声明时包含了本地初始值，则父组件的传参变为可选。
3. `@Prop` 装饰器不能在 `@Entry` 装饰的自定义组件（根组件）中使用。

## 变量的传递/访问规则

1. **父组件同步至子组件**：父组件中绑定给 `@Prop` 的变量发生变化时，会同步更新子组件的 `@Prop` 变量，并覆盖子组件本地对该变量的修改。
2. **子组件向父组件**：子组件对 `@Prop` 变量的修改是本地的，**无法改变**父组件对应的状态变量。
3. **父组件传入规则**：父组件中支持使用常规变量和下列装饰器装饰的变量：`@State`、`@Link`、`@Prop`、`@Provide`、`@Consume`、`@ObjectLink`、`@StorageLink`、`@StorageProp`、`@LocalStorageLink` 和 `@LocalStorageProp`。
4. **自身初始化规则**：自身 `@Prop` 装饰的变量支持初始化子组件的常规变量以及下列装饰器装饰的变量：`@State`、`@Link`、`@Prop`、`@Provide`。
![arkui_state_relationship](arkui_prop_relationship.png)

## 支持的类型

- **简单类型**：当装饰的数据类型为 boolean、string、number、enum 类型时，可以直接观察到数值的变化。
- **数组**：当装饰的对象是 Array 时，可以观察到 Array 整体的赋值及数组元素的赋值。调用 Array 的下述方法更新数据也能同步观察到变化：push, pop, shift, unshift, splice, copyWithin, fill, reverse, sort。
- **Map**：可以观察到 Map 整体的赋值。调用 Map 的下述方法更新 Map 值也能同步观察到变化：set, clear, delete。
- **Set**：可以观察到 Set 整体的赋值。调用 Set 的下述方法更新 Set 值也能同步观察到变化：add, clear, delete。
- **Date**：可以观察到 Date 的赋值。调用 Date 的下述方法更新 Date 属性也能同步观察到变化：setFullYear, setMonth, setDate, setHours, setMinutes, setSeconds, setMilliseconds, setTime, setUTCFullYear, setUTCMonth, setUTCDate, setUTCHours, setUTCMinutes, setUTCSeconds, setUTCMilliseconds。
- **类或 Object**：可以观察到自身的赋值和属性赋值的变化，**即 Object.keys(observedObject) 返回的所有属性**。需要注意的是，`@Prop` 在父组件向子组件同步深层对象时采用**深拷贝**机制（Deep Copy），深层嵌套（建议不超过 5 层）或复杂对象深拷贝会有一定性能开销。

 ```ts
// 声明Person类
class Person {
  public value: string;

  constructor(value: string) {
    this.value = value;
  }
}

// 声明Model类
class Model {
  public value: string;
  public name: Person;

  constructor(value: string, person: Person) {
    this.value = value;
    this.name = person;
  }
}

@Component
struct PropChildDemo {
  @Prop title: Model;

  build() {
    Column() {
      Text(`子组件 title:${this.title.value} name:${this.title.name.value}`)
      Button("子组件本地修改@Prop变量").onClick(() => {
        this.title = new Model('Child Modified', new Person('Child'));
      }) // 仅子组件UI更新，不更新父组件
      Button("子组件修改@Prop属性").onClick(() => {
        this.title.value = 'Child Property';
      }) // 仅子组件UI更新，不更新父组件
    }
  }
}

@Component
export struct PropClassOrModelDemo {
  @State title: Model = new Model('Hello', new Person('World'));

  build() {
    Column() {
      Text(`父组件 title:${this.title.value} name:${this.title.name.value}`)
      PropChildDemo({ title: this.title })
      
      Button("父组件修改@State变量").onClick(() => {
        this.title = new Model('Hi', new Person('ArkUI'));
      }) // ✅ 父组件和子组件UI均更新，子组件本地修改被覆盖
      Button("父组件修改@State属性").onClick(() => {
        this.title.value = '你好';
      }) // ✅ 父组件和子组件UI均更新，子组件本地修改被覆盖
    }
  }
}
 ```

## 例子

>[!example] 单向同步与 ForEach 配合时的组件复用行为
>
> 子组件允许本地修改 `@Prop` 变量，但在列表渲染 `ForEach` 场景下，如果更新父组件数组，结合 Diff 算法与组件复用，会导致 `@Prop` 本地修改产生特殊的保留逻辑。

```ts
@Component  
struct PropSimpleDemo {  
  @Prop value: number = 1;  
  
  build() {  
    Row() {  
      Button("-1").onClick(() => {  
        this.value -= 1;  
      })  
      Text(`${this.value}`).borderColor(Color.Red).borderRadius(2).borderWidth(1)  
      Button("+1").onClick(() => {  
        this.value += 1;  
      })  
    }  
  }  
}

@Component  
export struct PropSimpleParentDemo {  
  @State value: number = 0;  
  @State arr: number[] = [1, 2, 3];  
  
  build() {  
    Column() {  
      PropSimpleDemo({ value: this.value })  
      Row() {  
        Button("-100").onClick(() => {  
          this.value -= 100;  
        })  
        Text(`${this.value}`).borderColor(Color.Red).borderRadius(2).borderWidth(1)  
        Button("+100").onClick(() => {  
          this.value += 100;  
        })  
      }  
  
      PropSimpleDemo({ value: this.arr[0] })  
      PropSimpleDemo({ value: this.arr[1] })  
      PropSimpleDemo({ value: this.arr[2] })  
      Divider().height(5)  
      
      ForEach(this.arr, (item: number) => {  
        PropSimpleDemo({ value: item })  
      }, (item: number) => item.toString()) // 指定 Key 生成规则  
      
      Text('replace entire arr')  
        .onClick(() => {  
          // 两个数组都包含项“3”。  
          this.arr = this.arr[0] === 1 ? [3, 4, 5] : [1, 2, 3];  
        })  
    }  
  }  
}
```

> [!note] 针对 ForEach 中 @Prop 的行为解析
> 当每个 `PropSimpleDemo` 实例在本地对 `value` 进行累加（例如累加到 `7`）时：
> 若为 `ForEach` 指定了唯一 Key（如 `item.toString()`），触发数组从 `[1, 2, 3]` 变为 `[3, 4, 5]` 时：
> 1. 根据 Diff 算法，数组项 `"3"` 在变化前后始终存在，因此其对应的 `PropSimpleDemo` 组件实例**不会被删除重新生成**，而是被移动复用至首位。
> 2. 原 `"1"` 和 `"2"` 对应的组件实例被删除，同时新建 `"4"` 和 `"5"` 的组件实例。
> 3. 由于 `"3"` 对应的组件实例被保留，且父组件中数组项 `"3"` 的数值未发生变化（未触发父组件向子组件的刷新覆盖），该组件中先前在本地修改后的值 `7` 将被保留。
> 4. 最终 `ForEach` 的渲染结果为：`"7"`（复用原"3"组件）、`"4"`（新建）、`"5"`（新建）。