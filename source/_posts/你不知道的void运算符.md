---
title: 你不知道的 void 运算符
date: 2024-01-27 18:26:00
tags:
---

`void`运算符，一个在 JavaScript 中很常见，但你却不一定很熟悉的运算符。它出现在各个地方，你可能在下面这些地方见到过：

- HTML的`a`标签里

  ```html
  <a href="javascript:void(0);">This is a link</a>
  ```

- JS表达式（尤其是 tsc 或者 babel 编译后的代码里）
  
  ```javascript
  var result = foo === null || foo === void 0 ? void 0 : foo.bar;
  ```

- 箭头函数
  
  ```javascript
  button.onclick = () => void doSomething();
  ```

那么`void`运算符的作用究竟是什么呢？

## 缘起

之所以突然聊这个，是因为在某次代码CR里讨论到了TypeScript的[可选链（Optional Chaining）](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining)语法（当然从[ES2020](https://262.ecma-international.org/11.0/#sec-optional-chains)开始，JS中也支持了这个语法）。

可选链大家应该都不陌生，在获取深层次对象属性的时候极其方便，比如

```typescript
const result = foo?.bar;
```

上面这行代码作用类似于下面这种写法：

```typescript
const temp = foo === null || foo === undefined ? undefined : foo.bar;
const result = temp;
```

虽然是这样说，但是作为一个凡事都要试一试的人来说，还是自己编译一下看看比较好，在用 tsc 试了一下之后，发现编译后的代码如下所示：

```javascript
var result = foo === null || foo === void 0 ? void 0 : foo.bar;
```

好像和上面的有点不一样，这里的代码里所有的`undefined`全部换成了`void 0`，这是为什么呢？

## 分析

其实从MDN上，我们很容易知道`void`运算符的作用：

> `void`运算符对给定的表达式进行求值，然后返回`undefined`。

即不论`void`运算符后面跟的是什么，这个运算符总是返回`undefined`。比如下面这些写法都会返回`undefine`：

- `void 0`
- `void(0)`
- `void 'hello'`
- `void (1+2)`
- `void(console.log('Hello world'));`

也就是说上面 tsc 编辑后的语法中`void 0`和`undefined`是等价的。

而之所以不直接使用`undefined`，是因为在早期的 JavaScript 版本中，`undefined`是可以被重新赋值的，而使用`void 0`，则可以获取到**`undefined`的原始值**。

我们来看个例子：

```javascript
var undefined = 1;
var foo;
console.log(foo === undefined); // false
console.log(foo === void 0);    // true
```

上面这段代码运行后，第一个`console`将会打印`false`，第二个将会打印`true`，这就是因为`undefined`被重新赋值为了`1`，不再是原始的值，而为了保证编译后代码的正确性和稳定性，一般都会使用`void 0`来获取原始的`undefined`值。

到这里我们就搞明白刚刚那个问题了，那除了这个场景，`void`运算符还有哪些作用呢？

## 作用

### 1. 获取undefined的原始值

这个上面的例子中已经说到了，这里就不再赘述。

### 2. 用在 JavaScript URI 中，以及阻止`a`标签的默认行为

当用户点击一个以`javascript:`开头的 URI 时，它会执行 URI 中的代码，然后**用返回的值替换页面内容**，除非返回的值是`undefined`。这个时候`void`运算符就派上了用场，比如：

```html
<html>
  <head>
    <title>Test</title>
  </head>
  <body>
    <a href="javascript:'zzzzz'">Click me</a>
    <div>12345</div>
  </body>
</html>
```

在这个例子里，点击`a`标签后，整个页面的内容，包括下面的`12345`，都会被替换为`zzzzz`。

如果使用了`void`，则不会这样：

```html
<a href="javascript:void('zzzzz');">这个链接点击之后不会做任何事情</a>
<a href="javascript:void(document.body.style.backgroundColor='green');">
  点击这个链接会让页面背景变成绿色。
</a>
```

所以，很多地方的`a`标签都会添加一个简单的`href="javascript:void(0)`来阻止`a`标签的默认行为，达到类似`preventDefault`的效果。

可能有小伙伴有疑问，直接不写`href`属性或者`href`设为空字符串不就行了吗？其实还是有差别的,可以看下面的示例，从上到下三个`a`标签分别为：

1. 没有`href`属性
2. `href`值为空
3. `href`值为`javascript: void(0);`

![image.png](https://static.youfindme.cn/blog/void_operator/different_a_link.png)

可以看到还是有一些区别的。

不过，目前除了`javascript: void(0);`比较常见之外，其他利用`javascript:`伪协议来执行 JavaScript 代码已经十分少见了，同时这种行为也是不太推荐的做法，推荐的做法是给对应的元素绑定事件处理器。

***PS：***

这里有一些历史背景在里面，在早期，各个浏览器有着不同的 DOM2 事件处理接口，对于阻止默认行为也有不同的实现，比如

- 在支持W3C标准的浏览器中
  
  ```javascript
  linkEle.addEventListener('click', function (event) {
    event.preventDefault(); // 阻止默认行为
  });
  ```

- 而在IE中则是
  
  ```javascript
  linkEle.attachEvent('onclick', function (event) {
    event.returnValue = false; // 阻止默认行为
  });
  ```

所以很多开发者为了省事，不想写两套兼容代码，就直接省事在`a`标签的`href`属性中添加`javascript: void(0);`来阻止默认行为了，之后点击行为的自定义处理再通过 DOM0 支持的事件处理器来处理：

```javascript
linkEle.onclick = function () {
  // do something...
}
```

这种做法流传很广，所以即便是在现代浏览器对 W3C 标准支持的比较好的今天，我们还是能在很多地方看到`javascript: void(0);`。

### 3. 用于立即执行函数表达式（IIFE）

在使用立即执行函数表达式时，如果像下面这样写，会得到一个报错：

```javascript
function myFn () {
  console.log('Hello, world!');
}(); // Uncaught SyntaxError: Unexpected token ')'
```

为了解决这个问题，有两种方式：

- 给函数体包一层圆括号：
  
  ```javascript
  (function () {
    console.log('Hello world');
  })();
  ```

- 或者使用void运算符：
  
  ```javascript
  void function() {
    console.log('Hello, world!');
  }();
  ```

不过话说回来，IIFE 多用于 ES6 之前实现模块化，在 ES6 引入了`import`、`export`之后，IIFE 在实际的开发中用到的场景越来越少了，是一种过时的技术，这里是为了讲解`void`运算符才使用的，大家可以只做了解即可。

### 4. 强制返回`undefined`（常见于箭头函数中）

在箭头函数中，允许在函数体不使用花括号来直接返回值，比如：

```javascript
button.onclick = () => doSomething();
```

但是这样做会把`doSomething`函数的执行结果作为返回值来返回给调用方，而如果`doSomething`函数的返回值从`undefined`变成了其他值，可能会造成预料之外的结果。

所以如果明确`onclick`不需要返回值时，可以通过增加`void`运算符来强制返回`undefined`，防止因为`doSomething`返回值的变化带来其他副作用：

```javascript
button.onclick = () => void doSomething();
```

当然了，这里除了这种方式外，也可以通过给箭头函数增加一对花括号来解决：

```javascript
button.onclick = () => {
  doSomething();
};
```

## 结语

上面这些就是`void`运算符用的比较多的场景了，如果还有需要补充的，欢迎在评论区留言~
