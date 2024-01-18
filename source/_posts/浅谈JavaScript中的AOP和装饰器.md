---
title: 浅谈JavaScript中的AOP和装饰器
date: 2021-06-27 21:21:27
tags:
---

> AOP在其他编程语言中应用较多，在js中却应用较少，由于最近参与的项目中涉及到性能打点上报等，开始探索和了解无侵入地监控或修改代码的方式，趁此机会深入学习了一下AOP与装饰器

### 一、什么是AOP

AOP（Aspect-Oriented Programming），即**面向切面编程**，是一种编程范式，与之对应的有OOP（Object-Oriented Programming）面向对象编程。在JavaScript开发中，面向对象编程不管是ES5中直接操作原型还是ES6+中的class语法糖，大家用的比较多也应该都挺熟悉，但是相对而言，AOP用的就没那么多了。

对于OOP，我们在开发过程中往往是对类和对象通过继承进行**纵向拓展**，但是在AOP中则更侧重于把对象、方法作为切面，对原对象、方法进行**无侵入**的**横向拓展**，下面我们以一些简单的例子和代码来讲一下AOP。

假设我们有一个按钮，绑定了一个点击事件处理函数`handleClick`，现在需要对按钮增加点击上报，很简单，只需要在`handleClick`中调用一下上报函数就可以了：

```javascript
const report = (message) => {
  console.log(`上报了信息：${message}`);
}
const handleClick = (e) => {
  // do something...
  report('一条信息')
}
```

如果以后又需要增加对这个函数运行时间的上报，那么就需要在函数开头和结尾各自计算一下时间。

但是想一下，这样做带来的坏处是显而易见的，一是**不利于拓展**，每次增加新的点都需要改一次`handleClick`函数，改的越多越不利于后期维护；二是**有悖于单一职责原则**，`handleClick`里处理了大量非按钮的逻辑，同样增加了维护难度。这时候AOP就是一个很好的解决办法了。

### 二、ES6+之前的实现

在ES5以及ES6+的装饰器出现之前，我们往往是这样做的（这里用了ES6的部分语法），通过修改`Function`构造函数的原型，给原型增加两个方法`before`和`after`，如下：

```javascript
Function.prototype.before = function (beforeFn, beforeFnArgs) {
  const self = this;
  return function () {
    beforeFn.call(this, beforeFnArgs, ...arguments);
    return self.apply(this, arguments);
  };
};

Function.prototype.after = function (afterFn, afterFnArgs) {
  const self = this;
  return function () {
    const ret = self.apply(this, arguments);
    afterFn.call(this, afterFnArgs, ...arguments);
    return ret;
  };
};
```

通过这两个方法，实现在任意一个函数执行前后，执行我们自己自定义的行为，以达**到装饰或拦截**原函数、对象行为的目的。现在，实现上面点击上报的方式变成了这样：

```javascript
const report = (message) => {
  console.log(`上报了信息：${message}`);
};

let handleClick = function (e) {
  // do something...
};

handleClick = handleClick.after(report, '一条信息');
```

和修改原型类似的，我们还可以用[高阶函数HOF](https://zh.wikipedia.org/wiki/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)来实现一个上报函数执行时间的函数（`before`和`after`本质上也是高阶函数，只不过是定义在`Function`原型上的高阶函数）：

```javascript
const reportExeTime = (fn) => {
  return function () {
    let beforeTime;
    return fn
      .before(() => {
        beforeTime = Date.now();
      })
      .after(() => {
        report(`${Date.now() - beforeTime}`);
      });
  };
};

handleClick = reportExeTime(handleClick);
```

假设之后还要增加其他和业务逻辑无关的埋点、统计、监控等逻辑，只需要通过类似的方式来装饰原函数即可，而无需修改原函数。通过这种方式，我们实现了对原函数无侵入的横向拓展，是一种比继承更具弹性的代替方案。

其实到这里，很多小伙伴已经意识到了，在前端开发中已经有很多地方已经用到过AOP这种思想了，例如React的高阶组件HOC、Redux的中间件等，都是这种思想的体现。

### 三、ES6+通过装饰器实现

上面说的是ES6+中装饰器出现前AOP的实现方式，然而装饰器（Decorators）的出现给我们提供了另一种更为优雅的实现方式（本质上和高阶函数类似），虽然到目前为止这个特性也还只是在[提案的第三阶段](https://github.com/tc39/proposal-decorators)，还未真正引入到ES标准中，但是通过Babel（> 7.1.0）或者使用[TypeScript](https://www.tslang.cn/docs/handbook/decorators.html)，就可以提前使用这个特性。

关于装饰器的定义和使用方法已经有很多资料讲解了，这里不做过多介绍，我们来看下用装饰器怎么实现上面的逻辑。很容易想到，这里需要的是**方法装饰**器，首先定义一个上报的方法装饰器`withReport`：

```javascript
const withReport = (target, keyName, descriptor) => {
  const originFn = descriptor.value;
  descriptor.value = function () {
    const ret = originFn.apply(this, arguments);
    console.log('上报了信息'); // 数据上报
  };
  return descriptor;
};

class MyClass {
  @withReport
  handleClick(e) {
    // do something...
  }
}
```

或者我们还可以仿照上面的`before`和`after`，分别实现一个方法前置和后置**装饰器工厂**来执行任意前置后置方法，如下：

```javascript
const before = (beforeFn, beforeFnArgs) => {
  return function (target, keyName, descriptor) {
    const originFn = descriptor.value;
    descriptor.value = function () {
      beforeFn.call(this, beforeFnArgs, ...arguments);
      return originFn.apply(this, arguments);
    };
    return descriptor;
  };
};

const after = (afterFn, afterFnArgs) => {
  return function (target, keyName, descriptor) {
    const originFn = descriptor.value;
    descriptor.value = function () {
      const ret = originFn.apply(this, arguments);
      afterFn.call(this, afterFnArgs, ...arguments);
      return ret;
    };
    return descriptor;
  };
};
```

这样我们可以方便地传入一个`report`函数来实现上报：

```javascript
const report = (message) => {
  console.log(`上报了信息：${message}`);
};

class MyClass {
  @after(report, '一条信息')
  handleClick(e) {
    // do something...
  }
}
```

对于后续增加的其他非业务相关逻辑，也可以通过多个装饰器组合来实现，如上报函数执行时间：

```javascript
const reportExeTime = (target, keyName, descriptor) => {
  const originFn = descriptor.value;
  descriptor.value = function () {
    const beforeTime = Date.now();
    const ret = originFn.apply(this, arguments);
    report(`${Date.now() - beforeTime}`);
    return ret;
  };
  return descriptor;
};

class MyClass {
  @after(report, '一条信息') // 这个装饰器后执行
  @reportExeTime // 这个装饰器先执行
  handleClick(e) {
    // do something...
  }
}
```

通过装饰器，我们更优雅地实现了AOP，对方法进行横向拓展，而且不会像之前那样直接修改`Function`原型（确实是一种不太好的方式，污染原型链），装饰器之所以可以做到这一点是因为它并不是在运行时改变类、属性、方法的行为，而是在代码编译的时候就修改了，所以本质上来讲，装饰器就是一个**在编译时运行的语法糖函数**。

### 四、 AOP与装饰者模式和职责链模式

上面提到了，我们可以通过AOP来拓展函数或对象的行为，可以达到**装饰或拦截**原函数、对象行为的目的，对应的有两个设计模式，分别是**装饰者模式**和**职责链模式**（详见《JavaScript设计模式与开发实践》），AOP在这两种模式中的应用有一个明显的区别，就是能否阻断/覆盖原函数、对象的行为。有一点要声明的是，AOP只是实现这两种设计模式的其中一种方式，并不是唯一的实现方式，这里只是探讨在两种模式中AOP的应用。

对于装饰者模式，上面的几个示例都是很好的例子，不管是修改原型实现还是装饰器实现，都是通过`before`和`after`这两个来横向（区别于纵向的继承）的拓展原函数的能力，这里的`before`和`after`都是**静默执行**的，即`before`和`after`里执行的函数都不会阻断后续函数的执行或者覆盖原函数的返回值。应用场景也比较多，如上报、计时、日志等等，这也是AOP应用的比较多的场景。

然而对于职责链模式，对于`before`和`after`的能力要求则变成了：`before`和`after`里执行的函数可以阻断后续函数的执行，或`before`和`after`里执行的函数的返回值可以覆盖原函数的返回值。对应的代码实现变成了类似于下面这种：

- 修改原型实现

```javascript
Function.prototype.before = function (beforeFn, beforeFnArgs) {
  const self = this;
  return function () {
    const beforeRet = beforeFn.call(this, beforeFnArgs, ...arguments);
    if (!beforeRet) {
      // 如果不合法则阻断后续操作，这里为了表示方便这样简写
      return beforeRet;
    }
    return self.apply(this, arguments);
  };
};

Function.prototype.after = function (afterFn, afterFnArgs) {
  const self = this;
  return function () {
    const ret = self.apply(this, arguments);
    if (ret) {
      // 返回值符合条件时才返回，这里为了表示方便这样简写
      return ret;
    }
    return afterFn.call(this, afterFnArgs, ...arguments); // 在原函数返回值不符合条件时，执行after后续函数，覆盖原函数返回值
  };
};
```

- 装饰器实现

```javascript
const before = (beforeFn, beforeFnArgs) => {
  return function (target, keyName, descriptor) {
    const originFn = descriptor.value;
    descriptor.value = function () {
      const beforeRet = beforeFn.call(this, beforeFnArgs, ...arguments);
      if (!beforeRet) {
        // 如果不合法则阻断后续操作，这里为了表示方便这样写
        return beforeRet;
      }
      return originFn.apply(this, arguments);
    };
    return descriptor;
  };
};

const after = (afterFn, afterFnArgs) => {
  return function (target, keyName, descriptor) {
    const originFn = descriptor.value;
    descriptor.value = function () {
      const ret = originFn.apply(this, arguments);
      if (ret) {
        // 返回值符合条件时才返回，这里为了表示方便这样简写
        return ret;
      }
      return afterFn.call(this, afterFnArgs, ...arguments); // 在原函数返回值不符合条件时，执行after后续函数，覆盖原函数返回值
    };
    return descriptor;
  };
};
```

这种也有很多应用场景，最常见的就是表单等的校验，如传入的值不合法，则阻断后续函数执行，或者在原函数返回值不符合要求时，则用after里函数的返回值来覆盖。

### 五、 结语

其实除了上面提到的，我们还可以拓展下思路，上面对于函数的修饰，都是需要项目中的每个开发者手动在方法、类等前面显式调用，那么对于一些基础、通用的修饰方法，能否在打包构建时，通过打包构建插件分析语法树自动地给代码里的特定方法、类增加修饰，这个值得我们进一步思考探索。
