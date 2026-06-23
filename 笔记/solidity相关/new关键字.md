# new关键字

```solidity
const aa = new xxx
```

`aa`是对象，`xxx`是一个**类**或者**构造函数**

`new`关键字既可以用于实例化一个类，也可以用于实例化一个构造函数。

实际上，类本质上也是一种特殊的构造函数。**如果**`xxx`**是一个类，则**`new xxx()`**会调用该类的构造函数来创建对象实例**

```solidity
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
 
  sayHello() {
    console.log(`Hello, my name is ${this.name} and I'm ${this.age} years old.`);
  }
}
 
function Car(brand, model, year) {
  this.brand = brand;
  this.model = model;
  this.year = year;
 
  this.start = function() {
    console.log(`${this.brand} ${this.model} is starting...`);
  };
 
  this.stop = function() {
    console.log(`${this.brand} ${this.model} is stopping...`);
  };
}
 
// 实例化一个Person对象
const person = new Person("Alice", 25);
person.sayHello();
 
// 实例化一个Car对象
const car = new Car("Toyota", "Camry", 2022);
car.start();
car.stop();
```

`new`**关键字做了以下几件事情：**

1. **创建一个空对象**：`new`关键字会创建一个空的JavaScript对象，这个对象会继承自构造函数的原型。
2. **设置对象的原型链**：新创建的对象会**继承**构造函数的原型对象。这意味着通过`new`关键字创建的对象可以访问构造函数原型上定义的属性和方法。
3. **将构造函数的作用域赋给新对象**：将构造函数中的`this`关键字指向新创建的对象，使得构造函数可以操作新对象。
4. **执行构造函数的代码**：调用构造函数，并传入相应的参数，以初始化对象的属性和状态。
5. **返回新对象**：如果构造函数没有显式返回一个对象，则`new`关键字将返回新创建的对象。如果构造函数有显式返回一个对象，则返回该对象。

所以，`const aa = new xxx（）` 看到这个就代表，xxx是一个类或者构造函数，aa是实例化对象。通过传参可以初始化xxx的属性，aa.bb可以调用xxx上的bb方法

（如果xxx是一个类，则new xxx()会调用该类的构造函数来创建对象实例。如果xxx是一个构造函数，则直接调用new xxx()也会创建对象实例）


> 更新: 2025-07-11 10:25:59  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/og2ilicq37c6sd9g>