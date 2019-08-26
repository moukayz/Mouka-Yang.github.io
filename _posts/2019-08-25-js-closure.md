---
title: Javascript closure
categories:
- Web


---

<!-- more -->

> A closure is the combination of a function and the lexical environment within which that function was declared.
> 闭包：一个函数及其被定义时所在的词法环境

## 初体验
考虑代码
```
function init() {
  var name = 'Mozilla'; // name is a local variable created by init
  function displayName() { // displayName() is the inner function, a closure
    alert(name); // use variable declared in the parent function
  }
  displayName();
}
init();
```
上例中，内部函数`displayName`不包含变量`name`，但因为__内部函数可以访问外部函数的变量__，因此当`displayName`执行时，由于自身不包含`name`变量，因此会使用外部函数`init`的变量`name`

## 闭包
考虑代码
```
function makeFunc() {
  var name = 'Mozilla';
  function displayName() {
    alert(name);
  }
  return displayName;
}

var myFunc = makeFunc();
myFunc();
```
该代码运行结果和上例相同。二者的区别在于后者的内部函数`displayName`是作为函数从外部函数返回后再被执行的，即先执行完外部函数`makeFunc`，再执行内部函数`displayName`
> 在如c、c++、java等语言中，变量的生命周期只存在于其被声明时的代码块内，离开代码块后，相应的变量便无法访问（若上述代码改为c语言，则在`makeFunc`函数外部是无法访问其内部变量`name`的）
> __但在Javascript，情况则完全不同__

上述代码成功运行的原因是内部函数`displayName`的声明构成了一个闭包（closure），即`displayName`函数本身，及其声明时所在环境中的所有局部变量（词法环境，lexical environment）。因此，当`myFunc`（`displayName`的引用）被调用时，它还拥有一个对其闭包的引用，即它仍可以访问其声明时所在环境中的所有局部变量（包括`name`），因此函数可以正常执行

__再考虑如下代码__
```js
function makeAdder(x) {
  return function(y) {
    return x + y;
  };
}

var add5 = makeAdder(5);
var add10 = makeAdder(10);

console.log(add5(2));  // 7
console.log(add10(2)); // 12
```
其中，`makeAdder`为外部函数，并返回了一个内部匿名函数，可以访问外部函数的变量（此处为`x`）
此处`makeAdder`又叫做函数工厂（function factory，生成函数的函数），而`add5`和`add10`均为闭包。当闭包函数执行时（即代码`return x+y`）会从其词法环境中获取变量`x`的值，从其函数的参数列表获取`y`的值，并进行计算和返回。
注意，`add5`和`add10`是两个闭包，二者共享一个函数定义，__但拥有各自独立的词法环境__（`add5`中的`x`为5，`add10`中的`x`为10，二者互不影响）

## 模拟私有成员
JavaScript中没有私有成员（private）的概念，但可以通过闭包来模拟这一行为。
考虑代码
```js
var counter = (function() {
  var privateCounter = 0;
  function changeBy(val) {
    privateCounter += val;
  }
  return {
    increment: function() {
      changeBy(1);
    },
    decrement: function() {
      changeBy(-1);
    },
    value: function() {
      return privateCounter;
    }
  };
})();

console.log(counter.value()); // logs 0
counter.increment();
counter.increment();
console.log(counter.value()); // logs 2
counter.decrement();
console.log(counter.value()); // logs 1
```
该代码通过一个匿名函数定义了变量`privateCounter`和内部函数`changeBy`，并返回了一个对象，该对象包含三个闭包函数：`increment`,`decrement`,value`，最后将该对象赋值给`counter`
注意：三个闭包函数__共享同一个词法环境__，词法环境包括变量`privateCounter`和函数`changeBy`

代码随后调用闭包函数`value`，返回变量`privateCounter`的值（0）
然后调用闭包函数`increment`，该函数内部调用`changeBy`使`privateCounter`的值加1
调用两次后，变量`privateCounter`的值为2
闭包函数`decrement`同理

上述代码创建了一个对象`counter`，其包含三个方法，可用于修改或读取变量`privateCounter`的值，并且通过这三个方法访问`privateCounter`变量。因此模拟出了私有成员`privateCounter`

## 闭包范围链
对于每一个闭包有三个环境：
- 局部环境（闭包函数内部）
- 外部环境（闭包函数的外层函数）
- 全局环境

考虑如下示例
```js
// global scope
var e = 10;
function sum(a){
  return function(b){
    return function(c){
      // outer functions scope
      return function(d){
        // local scope
        return a + b + c + d + e;
      }
    }
  }
}

console.log(sum(1)(2)(3)(4)); // log 20
```
上例中，所有内部函数（闭包）可访问其所有外层函数（并不局限于其声明时所在的外部函数），例如最内层闭包函数可访问其所有外层函数的变量（a,b,c,e）
## 通过循环创建闭包
### 错误使用
考虑代码
```js
arr = []
for(var i = 1; i < 4; i++){
    arr.push(function(){
		return i
	})
}
arr.forEach(function(item) {
		console.log(item())
})
```
上例通过循环创建了3个闭包函数，并依次放入数组`arr`中，然后依次输出这三个闭包函数的执行结果。
*输出是 `1,2,3` 吗？*，**不，上例只会输出 `4,4,4`**
那么原因何在？
因为上述代码中，这三个闭包函数是共享一个词法环境的，所以三个函数引用的都是同一个`i`，而经过循环后`i`的值为4，所以三个闭包函数执行时都会引用`i`的当前值，即4，于是输出`4,4,4`
### 解决方案
**1. 更多的闭包**
考虑更改代码为
```js
function store(val){
	return function() {
		return val
    }
}
arr = []
for(var i = 1; i < 4; i++){
    arr.push(store(i))
}
arr.forEach(function(item) {
        console.log(item())
})
```
上述代码定义了一个函数工程`store`，其返回了一个闭包函数。即当`store(i)`执行时，返回的闭包函数的词法环境中只包括参数`val`，即`i`的**循环时的当前值**
代码可正确输出`1,2,3`
也可通过匿名函数的方式实现上述功能，如
```js
arr = []
for(var i = 1; i < 4; i++){
    arr.push((function(val){
        return function(){
            return val
        }
    })())
}
arr.forEach(function(item) {
        console.log(item())
})
```
**2. 使用`let`关键字**
考虑代码
```js
arr = []
for(let i = 1; i < 4; i++){
    arr.push(function(){
        return i
    })
}
arr.forEach(function(item) {
        console.log(item())
})
```
上述代码与错误示范的唯一区别就是将`for`循环中的`var`改成了`let`，由于`let`定义的变量的作用域为块作用域（`var`为函数级作用域），因此通过`let`定义的变量`i`的作用域仅为for循环中的一次循环，所以上例中创建的闭包的词法环境仅包含一次循环中的变量`i`（注意，错误示范中闭包的词法环境包含的是全局变量`i`），且每次循环创建的闭包包含的`i`值不同（达到了和使用多个闭包相同的作用），因而可以正确输出