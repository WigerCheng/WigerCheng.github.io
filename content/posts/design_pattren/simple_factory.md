---
date: "2025-08-20T15:27:58+08:00"
title: 简单工厂模式
---

# 简单工厂模式

* [图解简单工厂模式](https://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/simple_factory.html)
* 定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使一个类的实例化延迟到其子类。

## 模块定义

> 简单工厂模式(Simple Factory Pattern)：又称为静态工厂方法(Static Factory Method)模式，它属于类创建型模式。在简单工厂模式中，可以根据参数的不同返回不同类的实例。简单工厂模式专门定义一个类来负责创建其他类的实例，被创建的实例通常都具有共同的父类。

## 模式结构

简单工厂模式包含如下角色：

- Factory：工厂角色

    工厂角色负责实现创建所有实例的内部逻辑

- Product：抽象产品角色

    抽象产品角色是所创建的所有对象的父类，负责描述所有实例所共有的公共接口

- ConcreteProduct：具体产品角色

    具体产品角色是创建目标，所有创建的对象都充当这个角色的某个具体类的实例。

```java
public class SimpleFactoryDemo {
    public static void main(String[] args) {
        Operation oper = OperationFactory.createOpreation("+");
        oper.numA = 1;
        oper.numB = 2;
        try {
            double result = oper.getResult();
            System.out.println(result);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

//工厂角色
class OperationFactory {
    public static Operation createOpreation(String operate) {
        Operation oper = null;
        switch (operate) {
        case "+":
            oper = new OperationAdd();
            break;
        case "-":
            oper = new OperationSub();
            break;
        case "*":
            oper = new OperationMul();
            break;
        case "/":
            oper = new OperationDiv();
            break;
        }
        return oper;
    }
}

//抽象产品角色
abstract class Operation {
    protected double numA;
    protected double numB;

    abstract double getResult() throws Exception;
}

//具体产品角色
class OperationAdd extends Operation {
    @Override
    double getResult() {
        return numA + numB;
    }
}

//具体产品角色
class OperationSub extends Operation {
    @Override
    double getResult() {
        return numA - numB;
    }
}

//具体产品角色
class OperationMul extends Operation {
    @Override
    double getResult() {
        return numA * numB;
    }
}

//具体产品角色
class OperationDiv extends Operation {
    @Override
    double getResult() throws Exception {
        if (numB == 0)
            throw new Exception("除数不能为0");
        return numA / numB;
    }
}
```