---
title: 关于JavaScript中this的软绑定
date: 2021-06-27 21:11:06
tags:
---
#### 首先，什么是软绑定？

所谓软绑定，是和硬绑定相对应的一个词，在详细解释软绑定之前，我们先来看看硬绑定。在JavaScript中，this的绑定是动态的，在函数被调用的时候绑定，它指向什么完全取决于函数在哪里调用，情况比较复杂，光是绑定规则就有默认绑定、隐式绑定、显式绑定、new绑定等，而硬绑定是显式绑定中的一种，通常情况下是通过调用函数的`apply()`、`call()`或者ES5里提供的`bind()`方法来实现硬绑定的。

#### 硬绑定有什么问题，为什么需要软绑定

上述三个方法好是好，可以按照自己的想法将函数的this强制绑定到指定的对象上（除了使用new绑定可以改变硬绑定外），但是硬绑定存在一个问题，就是会降低函数的灵活性，并且在硬绑定之后无法再使用隐式绑定或者显式绑定来修改this的指向。

在这种情况下，被称为软绑定的实现就出现了，也就是说，通过软绑定，我们希望this在默认情况下不再指向全局对象（非严格模式）或`undefined`（严格模式），而是指向两者之外的一个对象（这点和硬绑定的效果相同），但是同时又保留了隐式绑定和显式绑定在之后可以修改this指向的能力。

#### 软绑定的具体实现

在这里，我用的是《你不知道的JavaScript 上》中的软绑定的代码实现：

```javascript
if (!Function.prototype.softBind) {
  Function.prototype.softBind = function (obj) {
    const fn = this;
    const args = Array.prototype.slice.call(arguments, 1);
    const bound = function () {
      return fn.apply(
        !this || this === (window || global) ? obj : this,
        args.concat.apply(args, arguments)
      );
    };
    bound.prototype = Object.create(fn.prototype);
    return bound;
  };
}
```

我们先来看一下效果，之后再讨论它的实现。

```javascript
function foo() {
  console.log('name: ' + this.name);
}

const obj1 = { name: 'obj1' },
  obj2 = { name: 'obj2' },
  obj3 = { name: 'obj3' };

const fooOBJ = foo.softBind(obj1);
fooOBJ(); //"name: obj1" 在这里软绑定生效了，成功修改了this的指向，将this绑定到了obj1上

obj2.foo = foo.softBind(obj1);
obj2.foo(); //"name: obj2" 在这里软绑定的this指向成功被隐式绑定修改了，绑定到了obj2上

fooOBJ.call(obj3); //"name: obj3" 在这里软绑定的this指向成功被硬绑定修改了，绑定到了obj3上

setTimeout(obj2.foo, 1000); //"name: obj1"
// 回调函数相当于一个隐式的传参，如果没有软绑定的话，这里将会应用默认绑定将this绑定到全局环境上，但有软绑定，这里this还是指向obj1
```

可以看到软绑定生效了。下面我们来具体看一下`softBind()`的实现。

在第一行，先通过判断，如果函数的原型上没有`softBind()`这个方法，则添加它，然后通过`Array.prototype.slice.call(arguments,1)`获取传入的外部参数，这里这样做其实为了函数柯里化，也就是说，允许在软绑定的时候，事先设置好一些参数，在调用函数的时候再传入另一些参数（关于函数柯里化大家可以去网上搜一下详细的讲解）最后返回一个`bound`函数形成一个闭包，这时候，在函数调用`softBind()`之后，得到的就是`bound`函数，例如上面的`var fooOBJ=foo.softBind(obj1)`。

在`bound`函数中，首先会判断调用软绑定之后的函数（如fooOBJ）的调用位置，或者说它的this的指向，如果`!this`（this指向undefined）或者`this===(window||global)`（this指向全局对象），那么就将函数的this绑定到传入`softBind`中的参数obj上。如果此时this不指向undefind或者全局对象，那么就将this绑定到现在正在指向的函数（即隐式绑定或显式绑定）。`fn.apply`的第二个参数则是运行`foo`所需要的参数，由上面的args（外部参数）和内部的arguments（内部参数）连接成，也就是上面说的柯里化。

其实在第一遍看这个函数时，也有点迷，有一些疑问，比如`var fn=this`这句，在`foo`通过`foo.softBind()`调用`softBind`的时候，fn到底指向谁呢？是指向foo还是指向softBind？我们可以写个demo测试，然后可以很清晰地看出fn指向什么：

```javascript
const a = 2;
function foo() {}
foo.a = 3;
Function.prototype.softBind = function () {
  const fn = this;
  return function () {
    console.log(fn.a);
  };
};
Function.prototype.a = 4;
Function.prototype.softBind.a = 5;

foo.softBind()(); // 3
Function.prototype.softBind()(); // 4
```

可以看出，fn（或者说this）的指向还是遵循this的绑定规则的，`softBind`函数定义在Function的原型`Function.prototype`中，但是JavaScript中函数永远不会“属于”某个对象（不像其他语言如java中类里面定义的方法那样），只是对象内部引用了这个函数，所以在通过下面两种方式调用时，fn（或者说this）分别隐式绑定到了`foo`和`Function.prototype`，所以分别输出3和4。后面的`fn.apply()`也就相当于`foo.apply()`。
