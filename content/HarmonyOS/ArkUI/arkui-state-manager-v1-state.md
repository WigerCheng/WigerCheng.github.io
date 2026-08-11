---
title: "@State_组件内状态"
tags:
  - "#StateV1"
  - HarmonyOS/ArkUI
---
## 原理

`@State` 装饰的变量可以使普通变量具备状态属性，当状态变量改变时，**会触发其直接绑定的UI组件渲染更新**。变量的生命周期与其所属自定义组件的生命周期相同，**即组件被移除该变量就过期**。

## 使用要求

1. `@State`修饰的是组件内部状态属性，是私有的，只能通过组件内部访问。
2. `@State`在声明时必须指定其类型并完成本地初始化。若需从父组件初始化，也可选择使用命名参数机制完成赋值。

## 变量的传递/访问规则

1. 如果父组件传入的是非undefined值将会覆盖本地初始值，否则使用@State的本地初始值。
2. 父组件传入的外部变量对@State初始化时，仅作为初始值，后续变量的变化不会同步至@State。
3. 父组件中支持使用常规变量和下列装饰器装饰的变量：@State、@Link、@Prop、@Provide、@Consume、@ObjectLink、@StorageLink、@StorageProp、@LocalStorageLink和@LocalStorageProp。
4. 自身@State装饰的变量支持初始化子组件的常规变量以及下列的装饰器装饰的变量：@State、@Link、@Prop、@Provide。
![arkui_state_relationship](arkui_state_relationship.png)

## 支持的类型

- **简单类型**：当装饰的数据类型为boolean、string、number类型时，可以直接观察到数值的变化。
- **数组**：当装饰的对象是Array时，可以观察到Array整体的赋值及数组元素的赋值，**数组项中==嵌套的属性==赋值无法观察**。调用Array的下述方法更新Array数据也能观察到变化：push, pop, shift, unshift, splice, copyWithin, fill, reverse, sort。
- **Map**：可以观察到Map整体的赋值。调用Map的下述方法更新Map值也能观察到变化：set, clear, delete。
- **Set**：可以观察到Set整体的赋值。调用Set的下述方法更新Map值也能观察到变化：add, clear, delete。
- **Date**：可以观察到Date的赋值。调用Date的下述方法更新Date属性也能观察到变化：setFullYear, setMonth, setDate, setHours, setMinutes, setSeconds, setMilliseconds, setTime, setUTCFullYear, setUTCMonth, setUTCDate, setUTCHours, setUTCMinutes, setUTCSeconds, setUTCMilliseconds。
- **类或Object**：可以观察到自身的赋值和属性赋值的变化，**即Object.keys(observedObject)返回的所有属性**。

 ```ts
 //现在有两个类，一个是Person，另一个是Model，其中Model有一个Person属性。
 
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
export struct StateClassOrModelDemo {  
  @State title: Model = new Model('Hello', new ClassA('World'));  
  
  build() {  
    Column() {  
      Text(  
        `title:${this.title.value} name:${this.title.name.value}`  
      )//title:Hello name:World  
      Button("对@State装饰变量的赋值。").onClick(() => {  
        this.title = new Model('Hi', new ClassA('ArkUI'));  
      })//✅title:Hi name:ArkUI  
      Button("对@State装饰变量的属性赋值。").onClick(() => {  
        this.title.value = '你好'  
      })//✅title:你好 name:ArkUI      
      Button("对@State装饰变量的嵌套属性赋值。").onClick(() => {  
        this.title.name.value = '世界'  
      })//❌title:Hello name:World  
    }  
  }}
 
 ```

## 例子

>[!example] 展开收起
>
> 通常在社交应用中有一个需求，如果文案过长可以展开收起，是否展开收起这个属性属于是组件内的状态属性。

```ts
@Component  
struct ExpandableNote {  
  @State isExpanded: boolean = false;  
  @State noteContent: string = "这是一条多行笔记内容。\n这是第二行。\n这是第三行，内容可以很长。\n这是第四行，用于演示多行文本的展开收起效果。";  
  
  build() {  
    Column() {  
      Text(this.isExpanded ? this.noteContent : (this.noteContent.substring(0, 20) + "..."))  
        .fontSize(16)  
        .fontColor(Color.Black)  
        .width('100%')  
        .textAlign(TextAlign.Start)  
        .maxLines(this.isExpanded ? undefined : 1)  
        .padding(10)  
        .border({ width: 1, color: Color.Gray })  
  
      Button(this.isExpanded ? "收起笔记" : "展开笔记")  
        .fontSize(14)  
        .margin({ top: 10 })  
        .onClick(() => {  
          this.isExpanded = !this.isExpanded;  
        })  
    }  
    .width('100%')  
    .padding(15)  
    .backgroundColor(Color.White)  
    .borderRadius(8)  
    .shadow({ radius: 4, color: Color.Gray, offsetX: 1, offsetY: 1 })  
  }  
}
```
