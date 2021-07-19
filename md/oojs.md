---
title: 面向对象的JavaScript
date: 2021-07-05
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "JS" #分类
tags:   #标签
	- JS
---
### 面向对象的JavaScript

#### 1.1 JavaScript中的对象

##### 1.1.1 对象字面量

> 对象字面量是由一对花括号和其中的键值对组成的。

##### 1.1.2 类

>  在Javascript中，只需要定义一个普通的函数，我们就能获取一个类。

###### 1.1.2.1 通过原型添加属性和方法

> Javascript中的每一个函数，都有一个prototype的属性，这个属性指向一个对象。
>
> 我们用new 来创建一个类的对象实例时，实例中所包含的属性和方法都来自prototype所指向的
>
> 这个对象。

```javascript
//定义一个名为Accommodation的构造函数
function Accommodation(){}

//为这个类添加属性
Accommodation.prototype.floors = 0;
Accommodation.prototype.rooms = 0;
Accommodation.prototype.sharedEntrance = false;
//为这个类添加方法
Accommodation.prototype.lock = function() {}
Accommodation.prototype.unlock = function() {}

```

##### 1.1.2.2 通过作用域添加属性和方法

> 函数体内定义的任何变量和函数，其作用域都限于该函数体内。
>
> 在所有嵌套函数中都可以访问定义在其父函数中的变量。

```javascript
//定义在任何函数之外的变量在全局作用域内，可以在任何位置访问
var myLibrary = {
  myName: "Evan"};
function doSomething(){
  var innerVar = 123;
  myLibrary.myName = 'Hello';
  function doSomethingElse() {
    innerVar = 1234;
  }
  
  doSomethingElse();
  alert(innerVar);//1234
}
```

> javascript开发者一般会结合使用prototype和this关键字来定义对象实例的属性和方法，其中前者用来定义方法，后者用来定义属性。
>
> 实现链式调用，只需要在类中的每个方法最后返回 this 关键字即可。

##### 1.1.2.3 继承

> javascript使用原型链来实现继承，就是prototype.

```javascript
function Accommodation(){}

Accommodation.prototype.lock = function() {}
Accommodation.prototype.unlock = function() {}

//定义一个构造函数，它将成为Accommodation的子类
function House(defaults) {
  defaults = defaults || {};
  this.floors = 2;
  this.rooms = defaults.rooms || 7;
}
//让House类继承Accommodatiob的所有内容 但是要把它的constructor改为自己的
House.prototype = new Accommodation();
House.prototype.constructor = House;

var house = new House();
```

##### 1.1.2.4 多态

> 在构造一个新的子类继承并扩展一个类的时候，你可能需要将某一个方法替换为同名的新方法，这就是多态。

```javascript
//定义父类
function Accommodation(){
  this.isLocked = false;
  this.isAlarmed = false;
}
//为所有的Accommodation添加方法，执行一些常见的操作
Accommodation.prototype.lock = function() {
  this.isLocked = true;
};

Accommodation.prototype.unlock = function() {
  this.isLocked = false;
};

Accommodation.prototype.alarm = function() {
  this.isAlarmed = true;
};

Accommodation.prototype.deactivateAlarm = function() {
  this.isAlarmed = false;
};

//定义一个子类House
function House(){}
House.prototype = new Accommodation();
House.prototype.lock = function() {
  //执行父类的lock方法，可以用函数call将上下文传递给该方法
  Accommodation.prototype.lock.call(this);
  //code here
  this.alarm()
}

```

#### 1.2 公有私有与受保护的属性和方法

> 在构造函数中通过var定义的变量其作用域局限于该构造函数内，在prototype上定义的方法无法访问这个变量。要想通过公有的方法来访问私有变量，则需要创建同时包含两个作用域的新作用域。为此，我们可以创建一个自我执行的函数，称为<strong><u>闭包</u></strong>。该函数完全包含了类的定义，包括私有变量与原型方法。
>
> <strong>闭包 可以理解为函数引用了外部的变量，非自己function内部定义的变量，并且把这个变量包了起来。</strong>

```javascript
//我们将类定义在一个自我执行的函数里，这个函数返回我们创建的类，并保存在一个变量
var Accommodation = (function(){

  function Accommodation() {}
  //此处定义私有变量，使用下划线前缀
  var _isLocked = false,
      _isAlarmed = false,
      _alarmMessage = 'Alarm activated';
  
  //私有的函数
  function _alarm(){
    _isAlarmed = true;
  }
  
  function _disableAlarm(){
    _isAlarmed = false;
  }
  
  //所有定义在原型上的方法都是公有的
  Accommodation.prototype.lock = function() {
    _isLocked = true;
  }
  
  Accommodation.prototype.unlock = function() {
    _isLocked = false;
  }
  
  //定义一个getter函数
 Accommodation.prototype.getIsLocked = function() {
   return _isLocked;
 }
  
 Accommodation.prototype.setAlarmMessage = function(message) {
   _alarmMessage = message;
 }
  return Accommodation;
}());

```