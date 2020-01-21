---
title: Js原型与原型链
categories: 
  - Javascript
tags: 
  - prototype
---
先看看javascript中一个常用的运算符: typeof
```js
function show(x) {
	console.log(typeof x); // undefined
	console.log(typeof 1); // number
	console.log(typeof 'a'); // string
	console.log(typeof true); // boolean

	console.log(typeof function(){}); // function
	
	console.log(typeof null); // object
	console.log(typeof [1, 'a', true]); // object
	console.log(typeof {a: 0, b: 'c'}); // object
	console.log(typeof new Number(1)); // object
}
```
其中上面的四种（undefined, number, string, boolean）属于简单的值类型，不是对象。剩下的几种情况——函数、数组、对象、null、new Number(1)都是对象。他们都是引用类型。
判断一个变量是不是对象非常简单。值类型的类型判断用typeof，引用类型的类型判断用instanceof。

**一切(引用类型)都是对象，对象是属性的集合，属性表示为键值对，键都为字符串**
```js
var obj = {
	name: ak,
	info: {
		tel: 123456,
		add: 'fj'
	},
	age: function(){
		return 18
	}
}
```

函数与对象的关系: 函数是一种对象，对象都是通过函数创建的
```js
var fn = function () {};
console.log(fn instanceof Object);  // true
```

每个函数都有一个属性: prototype，该属性的值是一个对象(属性的集合)，该对象默认只有一个constructor属性指向这个函数本身
```js
function Fn () {};
Fn.prototype.constructor === Fn // true
```
你可以在自己自定义的方法的prototype中新增自己的属性
```js
function Fn() { }
Fn.prototype.name = 'XYM';
Fn.prototype.getYear = function () {
    return 2000;
};

var fn = new Fn();
console.log(fn.name); // XYM
console.log(fn.getYear()); // 2000
```
Fn是一个函数，fn对象是从Fn函数new出来的，这样fn对象就可以调用Fn.prototype中的属性。
**每个对象都有一个隐藏的属性“__proto__”，这个属性指向创建这个对象的函数的prototype。即：fn.__proto__ === Fn.prototype**

那么存在两个疑问:
1. Object.prototype也是一个对象，它的__proto__指向哪里

自定义函数的prototype对象本质上就是和 var obj = {} 是一样的，都是被Object创建，所以它的__proto__指向的就是Object.prototype。
但是**Object.prototype确实一个特例，它的__proto__指向的是null**

2. 函数也是对象，函数也有__proto__吗

首先函数都是由Funtion函数创建出来的
自定义函数Foo.__proto__指向Function.prototype，Object.__proto__指向Function.prototype
Function函数也是对象，它的__proto__指向Function.prototype
**即Function是被自身创建的。所以它的__proto__指向了自身的Prototype**

Function.prototype指向的对象，它的__proto__也指向Object.prototype
因为Function.prototype指向的对象也是一个普通的被Object创建的对象，所以也遵循基本的规则。

总结：
函数有prototype属性，该属性是一个对象
对象有__proto__属性，该属性指向创建该对象函数的prototype属性
函数是一种特殊的对象，只要是对象就有__proto__属性，只要是函数就有prototype属性
函数是由Function创建，它的__proto__属性指向Function.prototype
函数的prototype是由Object创建，它的__proto__属性指向Object.prototype
Object.prototype.__proto__指向null
Object.__proto__指向Function.prototype
Function.prototype.__proto__指向Object.prototype
Function.__proto__指向Function.prototype

对于值类型，你可以通过typeof判断，string/number/boolean都很清楚，但是typeof在判断到引用类型的时候，返回值只有object/function，你不知道它到底是一个object对象，还是数组，还是new Number等等。
这个时候就需要用到instanceof。例如：
```js
function Foo(){}
var f1 = new Foo();
console.log(f1 instanceof Foo); // true
console.log(f1 instanceof Object); // true
```
instanceof运算符的第一个变量是一个对象，暂时称为A；第二个变量一般是一个函数，暂时称为B。
instanceof的原理是检查右边构造函数的prototype属性，是否在左边对象的原型链上。
```js
console.log(Object instanceof Function); // true
console.log(Function instanceof Object); // true
console.log(Function instanceof Function); // true
```
instanceof表示的就是一种继承关系，或者原型链的结构。

> 参考 [https://www.cnblogs.com/wangfupeng1988/p/3977987.html](https://www.cnblogs.com/wangfupeng1988/p/3977987.html)