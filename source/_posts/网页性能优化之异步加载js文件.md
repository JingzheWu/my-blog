---
title: 网页性能优化之异步加载js文件
date: 2021-06-27 21:01:37
tags:
---
一个网页的有很多地方可以进行性能优化，比较常见的一种方式就是异步加载js脚本文件。在谈异步加载之前，先来看看浏览器加载js文件的原理。
> 浏览器加载 JavaScript 脚本，主要通过`<script>`元素完成。正常的网页加载流程是这样的。
> 1. 浏览器一边下载 HTML 网页，一边开始解析。也就是说，不等到下载完，就开始解析。
> 2. 解析过程中，浏览器发现`<script>`元素，就暂停解析，把网页渲染的控制权转交给 JavaScript 引擎。
> 3. 如果`<script>`元素引用了外部脚本，就下载该脚本再执行，否则就直接执行代码。
> 4. JavaScript 引擎执行完毕，控制权交还渲染引擎，恢复解析 HTML 网页。
>
> 加载外部脚本时，浏览器会暂停页面渲染，等待脚本下载并执行完成后，再继续渲染。原因是 JavaScript 代码可以修改 DOM，所以必须把控制权让给它，否则会导致复杂的线程竞赛的问题。

上面所说的，就是我们平时最常见到的，将`<script>`标签放到`<head>`中的做法，这样的加载方式叫做**同步加载**，或者叫阻塞加载，因为在加载js脚本文件时，会阻塞浏览器解析HTML文档，等到下载并执行完毕之后，才会接着解析HTML文档。如果加载时间过长（比如下载时间太长），就会造成浏览器“假死”，页面一片空白。而且，放在`<head>`中同步加载的js文件中不能对DOM进行操作，否则会产生错误，因为这个时候HTML还没有进行解析，DOM还没有生成。由此看来，同步加载带来的体验往往并不好。

下面我们来看几种异步加载的方式。
1. **将`<script>`标签放到`<body>`底部**

    严格来说，这并不算是异步加载，但是这也是常见的通过改变js加载方式来提升页面性能的一种方式，所以也就放到这里来说。

    将`<script>`放到`<body>`底部，解决上上面说到的几个问题，一是不会造成页面解析的阻塞，就算加载时间过长用户也可以看到页面而不是一片空白，而且这时候可以在脚本中操作DOM。
2. **`defer`属性**

    通过给`<script>`标签设置`defer`属性，将脚本文件设置为延迟加载，当浏览器遇到带有`defer`属性的`<script>`标签时，会再开启一个线程去下载js文件，同时继续解析HTML文档，等等HTML全部解析完毕DOM加载完成之后，再去执行加载好的js文件。

    这种方式只适用于引用外部js文件的`<script>`标签，可以保证多个js文件的执行顺序就是它们在页面中出现的顺序，但是要注意，添加`defer`属性的js文件不应该使用`document.write()`方法。
3. **`async`属性**

    `async`属性和`defer`属性类似，也是会开启一个线程去下载js文件，但和`defer`不同的时，它会在下载完成后立刻执行，而不是会等到DOM加载完成之后再执行，所以还是有可能会造成阻塞。
    
    同样的，`async`也是只适用于外部js文件，也不能在js中使用`document.write()`方法，但是对多个带有`async`的js文件，它不能像defer那样保证按顺序执行，它是哪个js文件先下载完就先执行哪个。
4. **动态创建`<script>`标签**

    可以通过动态地创建`<script>`标签来实现异步加载js文件，例如下面代码：
    ```
    (function(){
        var scriptEle = document.createElement("script");
        scriptEle.type = "text/javasctipt";
        scriptEle.async = true;
        scriptEle.src = "http://cdn.bootcss.com/jquery/3.0.0-beta1/jquery.min.js";
        var x = document.getElementsByTagName("head")[0];
        x.insertBefore(scriptEle, x.firstChild); 
    })();
    ```
    或者
    ```
    (function(){
        if(window.attachEvent){
            window.attachEvent("load", asyncLoad);
        }else{
            window.addEventListener("load", asyncLoad);
        }
        var asyncLoad = function(){
            var ga = document.createElement('script'); 
            ga.type = 'text/javascript'; 
            ga.async = true; 
            ga.src = ('https:' == document.location.protocol ? 'https://ssl' : 'http://www') + '.google-analytics.com/ga.js'; 
            var s = document.getElementsByTagName('script')[0]; 
            s.parentNode.insertBefore(ga, s);
        }
    })();
    ```
    上面两种方法中，第一种方式执行完之前会阻止onload事件的触发，因为添加了`async`属性的异步脚本一定会在页面的load事件前执行，而现在很多页面的代码都在onload时还执行额外的渲染工作，所以还是会阻塞部分页面的初始化处理。第二种则不会阻止onload事件的触发。
    
    这里要简要说明一下`window.DOMContentLoaded`和`window.onload`这两个事件的区别，前者是在DOM解析完毕之后触发，这时候DOM解析完毕，JavaScript可以获取到DOM引用，但是页面中的一些资源比如图片、视频等还没有加载完，作用同jQuery中的ready事件。后者则是页面完全加载完毕，包括各种资源。

说完了这几种常见的异步加载js脚本的方式，再来看最后一个问题，什么时候用`defer`，什么时候用`async`呢？一般来说，两者之间的选择则是看脚本之间是否有依赖关系，有依赖的话应当要保证执行顺序，应当使用`defer`没有依赖的话使用`async`，同时使用的话`defer`失效。要注意的是两者都不应该使用`document.write()`，这个导致整个页面被清除重写。

下面一幅图表明了同步加载以及`defer`、`async`加载时的区别，其中绿色线代表 HTML 解析，蓝色线代表网络读取js脚本，红色线代表js脚本执行时间：
![js-load](https://my-cos-1254464911.cos.ap-guangzhou.myqcloud.com/js-load/js-load.jpg)
