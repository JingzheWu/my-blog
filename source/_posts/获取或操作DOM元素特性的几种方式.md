---
title: 获取或操作DOM元素特性的几种方式
date: 2021-06-27 21:10:18
tags:
---
1. 通过元素的属性

    可以直接通过元素属性获取或操作特性，但是只有公认的特性（非自定义的特性），例如`id`、`title`、`style`、`align`、`className`等，注意，因为在ECMAScript中，`class`是保留字（在ES6中成了关键字），所以就不能再用`class`这个属性名来表示元素的CSS类名了，只能换成`className`。
2. 通过`getAttribute()`、`setAttribute()`、`removeAttribute()`

    也可以通过这三个DOM方法来操作DOM元素的特性，例如

    ```javascript
    const div = document.getElementById("my-div");
    div.getAttribute("id");                 // 获取id
    div.getAttribute("title");              // 获取title
    div.getAttribute("class");              // 获取元素的CSS类名，因为是传参数给getAttribute函数，所以可以用class
    div.getAttribute("dat-my-attribute");   // 获取自定义特性
    div.setAttribute("id","anotherId");     // 设置id
    div.removeAttribute("title");           // 删除title
    ```

    从上面可以看出来，用这种方法，不仅可以获取到公认的特性，也可以获取自定义的特性。不过有两类特殊的特性，在通过属性获取和通过方法获取时获取到的却不一样，一类是`style`，通过属性访问获取到的是一个对象，通过`getAttribute`获取到的是CSS文本；另一类就是事件处理程序如`onclick`，通过属性获取，得到的是一个函数，而通过`getAttribute`获取得到的则是相应函数代码的字符串。
3. 通过元素的`attributes`属性

    Element类型的DOM元素都有一个`attributes`属性，如`div.attributes`，这样获取到的是一个NamedNodeMap，是一个动态的集合，和数组类似，也有`length`属性、可以通过下标访问遍历等,通常用途就是遍历元素特性，里面存放的是多个Att节点，每个节点的`nodeName`就是特性名称，`nodeValue`就是特性的值。它有一些方法，如下：
    - `getNamedItem(name)`：返回`nodeName`为name的节点
    - `setNamedItem(node)`：向集合中插入一个Attr节点
    - `removeNamedItem(name)`：删除集合中`nodeName`为name的节点
    - `item(pos)`：返回位于数字pos位置的节点
    例如要获取id，有如下代码

    ```javascript
    const div = document.getElementById("my-div");
    div.attributes.getNamedItem("id").nodeValue;
    div.attributes["id"].nodeValue;                 //后两行代码都可以获取到id，下面为简写形式
    ```

    从上面可以看出，这种方式最麻烦，所以平时基本不用，一般在遍历元素的特性时才会用到。
上面三种方式中，方式1是最常使用的，其次是2，最后才是第三种方式。
