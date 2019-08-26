---
title: Javascript 原型与继承
categories:
- Web
---

<!-- more -->

> 简单来说，一个对象的原型表示对象由该原型继承而来

## 简介
`foo.__proto__`表示对象`foo`继承的原型对象，任何对象均有该属性
`func.prototype`表示当函数作为被继承对象时，函数被继承的那部分。因此该属性只有拥有
例如：
```javascript
var a = new String() // 创建对象 a
a instanceof String // true，a继承于 String
typeof String // 'function' ，String类型为 function
a.__proto__ == String.prototype // true， a的原型为String.prototype
```
这样，即便以后String对象自身的属性改变，也不会影响到对象a
```js
String.add = ()=>{console.log} // String对象增加属性 add
a.add() //Uncaught TypeError: a.add is not a function
```
但如果改变`String.prototype`的属性，则会影响a（因为`a.__proto__`可看作指向`String.prototype`的指针）
```js
String.prototype.foo = ()=>{console.log(1)}
a.foo()	// print 1
```
## 原型链
当继承次数增多时，即形成原型链（prototype chain）
考虑示例
```js
var o = function() {
	this.foo = 1
	this.bar = 2
}
var a = new o()
```
对象a的原型链如下
`a =>  o.prototype => Object.prototype => null` （ __注意__ ，任何原型链的最终原型均为`null`）
### 属性继承
当需要访问对象的某一个属性时，会根据该对象的原型链按顺序查找该属性
继续上例
```js
o.prototype.bar = 3 // 修改 o.prototype.bar
o.prototype.c = 4 // 增加 o.prototype.c 

// 访问对象 a 的属性
console.log(a.foo) // print 1， 首先查找对象a的自身属性，找到 foo，输出

console.log(a.bar) // print 2，首先查找对象a的自身属性，找到bar，输出；
			// a.__proto__（即 o.prototype）也包含 bar，但不会被访问

console.log(a.c) // 4， 首先查找对象a的自身属性，无；再查找 a.__proto__的属性（即 o.prototype），找到 c，输出

console.log(a.unknown) // undefined，首先查找对象a的属性，无；再查找 a.__proto__的属性，无；再查找a.__proto__.__proto__的属性（即 Object.prototype），无；再查找a.__proto__.__proto__.__proto__，为 null，查找终止，返回 undefined

// {foo:1, bar:2} => {bar:3, c:4} => Object.prototype => null
```
### 方法继承
javascript中的函数可作为对象中的属性使用，和其他类型的属性相同（通过原型链查找以及属性覆盖）
但当被继承函数中包含`this`时，`this`指向的是该函数调用时所绑定的对象，而不是该函数属性所在的对象
例如，定义如下对象
```js
let o = {
	a = 1
	f : function() {
	console.log(this.a)
	}
}
console.log(o.f())	// print 1
```
然后通过`Object.create`创建一个派生类
```js
let m = Object.create(o) // 此时 m.__proto__ = o
m.a = 5 // 修改属性a
console.log(m.f())	// print 5，函数f中的 this 此时指向 m
```

## 对象创建方法汇总
### 1.语法构造
```js
var o = {a: 1}; // 普通对象

// o.__proto__ = Object.prototype
// o ---> Object.prototype ---> null


var b = ['yo', 'whadup', '?']; // 数组对象

// b.__proto__ = Array.prototype 
// b ---> Array.prototype ---> Object.prototype ---> null

function f() {
  return 2;
}

// Functions inherit from Function.prototype 
// (which has methods call, bind, etc.)
// f ---> Function.prototype ---> Object.prototype ---> null
```
### 2.通过构造函数
```js
function Graph() {
  this.vertices = [];
  this.edges = [];
}

Graph.prototype = {
  addVertex: function(v) {
    this.vertices.push(v);
  }
};

var g = new Graph();
// 当用new func() 创建对象时，大致步骤为：
// 1. 将 Graph() 作为 g的构造函数调用，因此g具有 vertices和edges属性
// 2. g.__proto__ = Graph.prototype
```
### 3. Object.create
```js
var a = {a: 1};  // 直接继承 Object 对象
// a ---> Object.prototype ---> null

var b = Object.create(a); // 继承 a 对象
// b ---> a ---> Object.prototype ---> null
console.log(b.a); // 1 (inherited)

var c = Object.create(b); // 继承 c 对象
// c ---> b ---> a ---> Object.prototype ---> null

var d = Object.create(null); // 直接继承 null，跳过Object
// d ---> null
console.log(d.hasOwnProperty); 
// undefined,  因为 该方法来自于Object对象
```
### 4. Class关键字
```js
'use strict';

class Polygon {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
}

class Square extends Polygon {
  constructor(sideLength) {
    super(sideLength, sideLength);
  }
  get area() {
    return this.height * this.width;
  }
  set sideLength(newLength) {
    this.height = newLength;
    this.width = newLength;
  }
}

var square = new Square(2);
```

### hasOwnProperty(prop)
该函数查询对象的自有属性是否包含prop，不会追溯原型链（唯一）
```js
console.log(g.hasOwnProperty('vertices'));
// true

console.log(g.hasOwnProperty('nope'));
// false

console.log(g.hasOwnProperty('addVertex'));
// false

console.log(g.__proto__.hasOwnProperty('addVertex'));
// true
```

