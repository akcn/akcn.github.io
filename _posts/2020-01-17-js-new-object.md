---
title: Js对象的创建
categories: 
  - Javascript
tags: 
  - new object
---
对象是一个容器，封装了属性（property）和方法（method）。

### 通过new创建对象
Js使用构造函数（constructor）作为对象的模板，构造函数就是一个普通的函数，但是有自己的特征和用法。
```js
var Vehicle = function(){
	this.price = 1000;
}
```
构造函数有两个特点:
1. 函数体内部使用了this关键字，代表了所要生成的对象实例。
2. 生成对象的时候，必须使用new命令。
new命令的作用，就是执行构造函数，返回一个实例对象。
```js
var v = new Vehicle();
v.price // 1000
```
如果调用构造函数忘记使用new，构造函数变成普通对象，内部的this指向全局对象，会造成意想不到的结果。
使用new命令时，它后面的函数依次执行下面的步骤。

1. 创建一个空对象，作为将要返回的对象实例。
2. 将这个空对象的原型，指向构造函数的prototype属性。
3. 将这个空对象赋值给函数内部的this关键字。
4. 开始执行构造函数内部的代码。
```js
function _new(/* 构造函数 */ constructor, /* 构造函数参数 */ params) {
  // 将 arguments 对象转为数组
  var args = [].slice.call(arguments);
  // 取出构造函数
  var constructor = args.shift();
  // 创建一个空对象，继承构造函数的 prototype 属性
  var context = Object.create(constructor.prototype);
  // 执行构造函数
  var result = constructor.apply(context, args);
  // 如果返回结果是对象，就直接返回，否则返回 context 对象
  return (typeof result === 'object' && result != null) ? result : context;
}

// 实例
var actor = _new(Person, '张三', 28);
```
Function.prototype.call，Function.prototype.apply用来绑定this的指向

`func.call(thisValue, arg1, arg2, ...)`
`func.apply(thisValue, [arg1, arg2, ...])`
apply方法的作用与call方法类似，唯一的区别就是，它接收一个数组作为函数执行时的参数
```js
var obj = {};

var f = function () {
  return this;
};

f() === window // true
f.call(obj) === obj // true

function add(a, b) {
	return a + b;
}
add.call(this, 1, 2) // 3
add.apply(this, [1, 2]) // 3
```

### Object.create()创建对象
JavaScript 提供了Object.create方法，从一个实例对象，生成另一个实例对象。
该方法接受一个对象作为参数，然后以它为原型，返回一个实例对象。该实例完全继承原型对象的属性。
```js
// 原型对象
var A = {
  print: function () {
    console.log('hello');
  }
};

// 实例对象
var B = Object.create(A);

Object.getPrototypeOf(B) === A // true
B.print() // hello
B.print === A.print // true
```
上面代码中，Object.create方法以A对象为原型，生成了B对象。B继承了A的所有属性和方法。
实际上，Object.create方法可以用下面的代码代替。
```js
if (typeof Object.create !== 'function') {
  Object.create = function (obj) {
    function F() {}
    F.prototype = obj;
    return new F();
  };
}
```
上面代码表明，Object.create方法的实质是新建一个空的构造函数F，然后让F.prototype属性指向参数对象obj，最后返回一个F的实例，从而实现让该实例继承obj的属性。

> 参考 [https://wangdoc.com/javascript/](https://wangdoc.com/javascript/)