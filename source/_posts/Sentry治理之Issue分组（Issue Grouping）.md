---
title: Sentry治理之Issue分组（Issue Grouping）
date: 2023-12-11 13:05:23
tags:
---


## 一、前言

Sentry大家应该都不陌生，即便没有使用过，也应该听过Sentry的大名。

作为一个实时事件日志监控平台，Sentry可以记录和聚合我们应用中的报错、打点等，不管是Sentry自动捕获的错误，还是我们主动上报的错误，都可以在Sentry提供的可视化平台看到，方便开发者及时发现、分析和排查应用中存在的问题。

但是在使用Sentry的过程中，我们发现了一些使用起来不那么方便的地方，这个就是我们今天要一起讨论的问题——Sentry Issue的分组（Issue Grouping）。

## 二、先看看什么是Sentry Issue

在Sentry中，每一条日志上报都是一个事件（Event）,在Sentry的Discover面板中，我们可以看到所有上报的Event，比如我这个项目：

![Discover面板](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/discover.png)

Event分为两种类型，Transaction和Error。

Transaction事件主要用于性能监控。它记录了一个请求或任务从开始到结束的完整生命周期，包括各种详细的性能数据，如请求的开始时间、结束时间、总耗时、各个阶段的耗时等。

Error事件主要用于错误跟踪。它记录了应用运行过程中发生的错误或异常，包括错误的类型、位置、堆栈跟踪等信息。

而**Issue就是Error类型的Event的聚合**，Sentry会把一些相似的Error进行聚合，合并成一个Issue，这样我们就可以看到某个特定Error发生的频率和趋势，而不仅仅是只能看到单个Error Event。

Sentry的Issue可以在Issues面板中看到，如下图所示：

![Issue面板](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/issues.png)

从这个面板可以看到某个Issue（即某个类型的Error），上报了几次，有多少用户遇到了这个Error，以及这个Error数量变化的趋势，帮助我们快速确认问题的严重程度和影响范围。

## 三、我们遇到啥问题了？

从上面的描述可以看到，Sentry把Error进行聚合，合并成一个个Issue，帮助我们查看某个类型Error的一些信息，看起来是挺好的。

但是在我们的项目里，Sentry好像并不是这么做的，比如下图：

![重复的Issue](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/repeat_issues.png)

说好的自动聚合分组呢？

同样的一个Error（或者是极其相似的Error）,并没有被聚合为同一个Issue，而是分到了不同的几个Issue里，并且这些Issue的名字几乎一摸一样，每个Issue还各自展示一个Event次数。

而且这个问题不止出现在某一种类型的Error上，几乎所有的Error上报都或多或少地存在这种问题，导致不能很好地分析某种Error的影响或者变化趋势。

而且有时候即便我们手动Ignore某个Issue，未来还是会不断地有新的这个Issue出现，或者我们像下面这样手动Merge两个Issue，也还是会源源不断地产生新的、没有被Merge进手动Merge的分组内的Issue。

![手动Merge](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/merge_issues.png)

## 四、研究下Sentry是怎么对Error分组的

在解决我们遇到的问题之前，还是要先了解下Sentry是怎么对Error进行分组的，知道原理才能着手解决。

看了下官方文档[Issue Grouping | Sentry Documentation](https://docs.sentry.io/product/data-management-settings/event-grouping/)，这才揭开了Sentry分组的面纱。

### 1. Sentry Issue的Fingerprint和分组

在Sentry中，有一个“指纹”的概念，Fingerprint，Fingerprint是标识Event的一种方式，每个Event（包括Error和Transaction）都有一个Fingerprint。

Sentry会根据某种规则，来给每一个Event生成Fingerprint，具有相同Fingerprint的Event会被Sentry分为一组，这就是Sentry分组的基本原则。

#### 1.1 如何在Sentry上查看一个Event的Fingerprint呢？

从Discover或者Issues列表中，随便点击一个进入Error详情（Transaction不行，下面会讲原因），点击查看这个Error对应原始JSON数据：

![查看原始JSON](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/issue_detail_json.png)

在原始JSON中搜索fingerprint字段，可以看到如下所示：

```json
{
  "fingerprint": [
    "{{ default }}"
  ]
}
```

一个`{{ default }}`，这说明使用的是Sentry默认规则生成的Fingerprint。如果是其他的规则，则会展示为其他的值。

#### 1.2 Event默认的Fingerprint生成规则

不同类型的Event，有不同的Fingerprint生成规则：

- Error类型：Error类型会基于这个Error的调用堆栈`Stack Trace`，异常类型`Exception`，和日志消息`message`，从这三个方面来生成Fingerprint
- Transaction类型：通过这个类型的Spans来生成，可以查看原始JSON数据中的`spans`字段

我们这次只讨论Error类型的Event Fingerprint生成规则。

首先，Sentry每个版本生成Fingerprint的默认规则可能会有一些差异，每次Sentry默认的Fingerprint生成规则变化了之后，Sentry都会发布一个新版本，所以Fingerprint生成规则变化了之后，不会影响已有的Event。

每次新建一个Project，都会自动使用目前最新版本的Fingerprint生成规则，如果想要现有的Project升级到最新的Fingerprint生成规则，需要在设置里手动修改，具体位置为：**Settings > Project > [Your Project] > Processing > Issue Grouping > Upgrade Grouping**.如下图：

![升级分组](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/upgrade_grouping.png)

所有版本的Fingerprint生成规则都是最先考虑`Stack Trace`，然后是`Exception`, 最后是`message`。

##### a. 按照Stack Trace分组

对于一个上报的Error Event，如果他的原始数据中有调用堆栈信息，就会完全根据调用堆栈来进行分组（即不考虑其他的），主要会使用下列信息：

- 模块名module
- 文件名（去除哈希值等之后的名字）filename
- 上下文，行号列号等信息

这里的堆栈信息只包括和当前Project有关的堆栈，和当前项目无关的堆栈信息不会用于分组。

堆栈信息可以在原始的JSON数据中的`exception.values`的`stacktrace`字段中看到，如下所示：

```json
{
  "exception": {
    "values": [
      {
        "type": "Error",
        "value": "xxx err",
        "stacktrace": {
          "frames": [
            {
              "function": "xxx",
              "module": "yyy",
              "filename": "zzz",
              "abs_path": "app:///aaa/bbb/ccc.js",
              "lineno": 10,
              "colno": 30,
              "in_app": true
            },
            {
              "function": "rrr",
              "module": "uuu",
              "filename": "bbb",
              "abs_path": "app:///ddd/eee/fff.js",
              "lineno": 20,
              "colno": 57,
              "in_app": true
            },
            // ......
          ]
        }
      }
    ]
  }
}
```

在`stacktrace`字段中有个`frames`，是一个数据，记录的是当前Error发生时的调用堆栈帧列表，数组中的每一项就是一个调用帧（frame），每一帧中都有如下信息：

```json
{
  "function": "xxx",
  "module": "yyy",
  "filename": "zzz",
  "abs_path": "app:///aaa/bbb/ccc.js",
  "lineno": 10,
  "colno": 30,
  "in_app": true
}
```

而Sentry就是根据这些调用栈的帧列表，来生成Fingerprint。即相同调用堆栈的错误，被认为是同一种类Error，会被归为同一组。

用这个方法来分组，一般来说效果都挺不错，但是如果出现下面这些情况，就会导致这个分组方法或者说分组规则，效果不那么好：

1. 代码经过混淆或者压缩（比如TS/JS代码经过Babel编译）
  由于混淆或者压缩之后，代码的变量名、函数名、代码结构等都会发生变化，即便对于同一个Error，不同版本的代码（比如两个release版本之间，或者两次不同的构建之间）的调用堆栈信息也会发生变化，导致Sentry认为这些是不同的Error，从而没有进行聚合分组。
  如果代码有混淆或者压缩，就需要上传Source Maps到Sentry，让Sentry通过原始的堆栈信息生成Fingerprint，来避免分组混乱。

2. 代码通过装饰器等引入了新的堆栈层级，也会导致调用堆栈发生变化。比如

```javascript
const decoratorFn = (target, keyName, descriptor) => {
  const originFn = descriptor.value;
  descriptor.value = function () {
    console.log('Before function execution');
    const ret = originFn.apply(this, arguments);
    console.log("After function execution");
    return ret;
  };
  return descriptor;
};
 
class MyClass {
  @decoratorFn
  handleClick(e) {
    console.log('Inside function');
  }
}
```

上面这个例子中，`myFunction`被`decoratorFn`装饰。当调用`myFunction`时，实际上是在调用`decoratorFn`返回的函数。因此，如果在这个过程中发生错误并生成堆栈信息，堆栈信息中将会包含`decoratorFn`返回的函数的调用，这就是通过引入新的堆栈层级来修改堆栈信息。

可以通过自定义的`beforeSend`来自定义删除或者修改这些堆栈信息（下面会讲到）。

##### b.按照Exception分组

如果上报的Error Event的数据中没有堆栈调用信息，但是有完整的`Exception`信息（这里的完整指的是`Exception`的type和value同时存在），就会按照`Exception`的type和value进行分组，但是这种分组可靠性比较低，因为错误的信息可能会改变。

这里的`Exception`指的就是各种Error，JS里的`Exception`就是各种类型的`Error`，比如`ReferenceError`、`TypeError`、`RangeError`、`SyntaxError`等。

- `Exception`的type就是Error的name，如果有自定义的name就是自定义的name，比如`DownloadFileError`，没有自定义的话默认就是各个类型名
- Exception的value指的就是err.message

比如访问一个未定义的变量，JS会抛出一个`ReferenceError`，type就是“ReferenceError”，value可能是“x is not defined”。

如果是下面的自定义name的Error：

```javascript
const err = new Error('This is a mock err msg');
err.name = 'MyMockError';
```

那么这个Error的type就是“MyMockError”，value就是“This is a mock err msg”。

这里的type和value也可以直接在Sentry上任意一个Error原始的JSON数据中看到，JSON中的`metadata`字段，记录了这个Error的type和value：

```json
{
  "metadata": {
    "display_title_with_tree_label": false,
    "filename": "app:///aaa/bbb/ccc.js",
    "function": "xxx",
    "title": "MyMockError: This is a mock err msg",
    "type": "MyMockError",
    "value": "This is a mock err msg"
  }
}
```

> ***问：***
什么情况下会出现存在Exception但是不包含堆栈信息？
>
> ***ChatGPT:***
在大多数情况下，当JavaScript抛出一个异常时，这个异常都会包含一个堆栈跟踪。堆栈跟踪记录了错误发生时的函数调用序列，这对于理解和调试错误是非常有用的。
> 然而，在某些情况下，异常可能不包含堆栈跟踪。例如：
>
> 1. 如果你手动创建并抛出一个错误，但没有提供堆栈跟踪，那么这个错误就不会有堆栈跟踪。例如，throw new Error()会创建一个包含堆栈跟踪的错误，但throw { message: 'error' }则不会。
> 2. 在某些JavaScript环境中，例如某些旧版本的浏览器或某些JavaScript引擎，可能不支持堆栈跟踪，或者支持的方式与标准不同。
>
> 3. 如果错误发生在异步代码中，并且这个错误没有被正确地捕获和处理，那么可能只有错误信息，没有堆栈跟踪。
>
> 4. 如果你的代码中有捕获错误并处理的逻辑，可能会修改或移除堆栈跟踪。

##### c. 兜底的分组

如果上面两种情况都没办法对Event进行分组，那么就会使用兜底的分组，即直接使用上报的时候收到的Event消息来分组。

#### 1.3 分析一下

到这里我们可以先分析一下，为什么我们的项目会出现上面说的问题了。

首先，我们的项目没有修改过任何和Event Fingerprint有关的设置，使用的是默认分组规则，即使用调用堆栈`Stack Trace`，异常类型`Exception`，和日志消息`message`来进行分组。而绝大部分都是使用调用堆栈进行分组。

我们的JS项目由于某种原因，在编译后没有把Source Maps上传到Sentry，导致代码的变量名、函数名、代码结构等在不同版本或者不同的构建记录后，都会发生变化，所以即便某个Issue被Ignore或者被手动Merge，到下一个版本，由于同一个Error的调用栈变化了，生成了完全不同的Fingerprint，导致没有被分为一组。

![混淆压缩后的代码的调用栈](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/mixed_call_stack.png)

> **💡注意：**
> 代码混淆之后并不是会让Sentry没办法对Error生成Fingerprint以及分组，即使代码被混淆和压缩，只要所有用户都使用的是同一份混淆和压缩后的代码，那么同一个地方的Error应该会生成相同的堆栈跟踪，Sentry应该能够正确地将这些错误分到同一组。
>
> 真正的问题在于多个版本或者多个构建之间，每次压缩混淆后的代码都不一样，从而导致不同版本直接Error分组混乱。

看来使用默认的Fingerprint生成规则不行了，至少在我们项目上传Source Maps之前不行。需要看下怎么自定义分组。

#### 1.4 自定义分组

首先，只有Error类型的Event支持自定义分组，Transaction类型的暂时无法自定义。这也是为什么上面说Transaction类型的Event，无法在原始JSON数据中看到fingerprint字段的原因，因为Transaction Event无法自定义，所以也就不会展示在JSON数据里。

对于Error类型的Event，从简单到复杂有以下4种方式来自定义分组：

1. 在Sentry Admin对应的项目Issues列表中，手动Merge
  手动合并（你认为是）相同的Issues，最简单，不需要修改任何设置和配置项。

2. 在Sentry Admin对应的项目设置中，设置自定义的Fingerprint Rules
  设置Fingerprint Rules，只影响新上报的的Event，不影响已经上报的Event。

3. 在Sentry Admin对应的项目设置中，设置自定义的Stack Trace Rules
  设置Stack Trace Rules，只影响新上报的的Event，不影响已经上报的Event。

4. 在使用Sentry SDK的本地项目里，使用SDK Fingerprinting
  在本地项目中，使用SDK上报之前，设置Event的Fingerprint。

下面我们一个个来看。

### 2. 手动合并Issue

在Sentry项目的Issues列表中，手动选择2或者更多个Issue，然后点击Merge，即可合并为一个分组。

![手动Merge](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/merge_issues.png)

需要注意的是，Sentry并不会根据我们如何手动Merge，来改变或者推断出任何新的分组规则，新产生的Issue还是会按照之前的规则来分组，然后根据放到我们手动Merge的Issue集合中。

这也解释了为什么我们项目中，每次手动Merge之后，还是会产生新的没有被进入Merge后的分组，因为“Sentry并不会根据我们如何手动Merge，来改变或者推断出任何新的分组规则”。

### 3. Stack Trace Rules

虽然按照Sentry官网的文档的说法，Stack Trace Rules要比Fingerprint Rules复杂一些，我们还是先来讲下Stack Trace Rules。

在比较旧的Sentry版本中，Stack Trace Rules也叫作Grouping Enhancements或者Custom Grouping Enhancements。

具体设置位置为：**Settings > Project > [Your Project] > Processing > Issue Grouping > Stack Trace Rules**。

或者旧版本中为：**Settings > Project > [Your Project] > General Settings > Custom Grouping Enhancements**。

![Stack Trace Rules](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/stack_trace_rules.png)

修改Stack Trace Rules会影响**输入到Stack Trace分组算法中的数据**。我们可以通过规则来改变哪些stack trace frames被视为"in-app"，这会影响Sentry如何将Issue分组。例如，我们可以将某些通常被视为"not in-app"的frames标记为"in-app"，这样它们就会被包含在分组算法中。

在自定义的Stack Trace Rules中，每一行都是一条规则。每条规则有匹配项（matcher）、表达式（expression），以及跟在后面的操作（action）组成：

```glob
matcher-name:expression other-matcher:expression ... action1 action2 ...
```

一条规则里可以有多个匹配表达式，后面也可以有多个action。这些action会在前面所有匹配表达式匹配的时候执行。

所有的规则会从上到下，对调用堆栈信息里的所有帧（Frames）执行。

如果要表达否定，那么就在matcher前加上一个感叹号`!`，某一行以`#`开头则表达这一行是注释。

下面是一些例子：

```glob
# mark all functions in the std namespace to be outside the app
family:native stack.function:std::*       -app

# mark all code in node modules not to be in app
stack.abs_path:**/node_modules/**         -app

# remove all generated javascript code from all grouping
stack.abs_path:**/generated/**.js         -group
```

由于Stack Trace Rules不是我们这次讨论的重点，这里就不太说太多了，更多详细的关于Matchers和Actions的信息，详见官方文档：[Matchers](https://docs.sentry.io/product/data-management-settings/event-grouping/stack-trace-rules/#matchers)，[Actions](https://docs.sentry.io/product/data-management-settings/event-grouping/stack-trace-rules/#actions)。

### 4. Fingerprint Rules

在比较旧的Sentry版本中，也叫作Server Side Fingerprinting（叫这个名字是为了和SDK Fingerprinting对应）。

具体设置位置为：**Settings > Project > [Your Project] > Processing > Issue Grouping > Fingerprint Rules**。

或者旧版本中为：**Settings > Project > [Your Project] > General Settings > Server Side Fingerprinting**。

![Fingerprint Rules](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/fingerprint_rules.png)

Fingerprint Rules的配置方式和Stack Trace Rules类似，只有语法上不同。但是和Stack Trace Rules不同的是，Fingerprint Rules允许我们直接指定一个Issue的Fingerprint，它会完全覆盖默认的分组规则。

可以理解为，**Stack Trace Rules更关注如何改变分组算法的输入数据**（比如翻转一些标志位，或者对调用栈做一些裁剪），而**Fingerprint Rules则直接指定了分组的结果**。

首先，Fingerprint Rules同样是每一行是一条规则。每一条规则的Matcher和Stack Trace Rules的语法规则也是一样的，并且都可以设置`!`来表示取反，以及设置`#`来注释。

```glob
# You can use comments to explain the rules.  Rules themselves follow the
# following syntax:
matcher:expression -> list of values
# The list of values can be hardcoded or substituted values.
```

Fingerprint Rules也是把一个Event从上到下进行匹配，每条规则都是对调用堆栈信息里的所有帧（Frames）执行，并且会把匹配到的第一条规则作为Event的Fingerprint。

不同的是，Stack Trace Rules的Matcher右侧是对Stack Trace Frames数据进行的一些操作，Fingerprint Rules的Matcher右侧直接就是需要指定的Fingerprint的值，可以是一些写死的**常量**，也可以是一些[内置的变量（Variables）](https://docs.sentry.io/product/data-management-settings/event-grouping/fingerprint-rules/#variables)。

下面的例子就是把Error类型的Event根据type和value进行分组：

```glob
# 把 DatabaseUnavailable 和 ConnectionError 这两种类型的 Error，都标记为 system-down
error.type:DatabaseUnavailable -> system-down
error.type:ConnectionError -> system-down

# 把 Error message 中，包含“connection error: ”的，都标记为 connection-error，同时把当时 Error 的 transaction 字段也拼接到 Fingerprint 中
error.value:"connection error: *" -> connection-error, {{ transaction }}
```

#### 4.1 Matchers

对于Matchers，Sentry允许使用[glob patterns](https://en.wikipedia.org/wiki/Glob_(programming))语法。Sentry包含了以下的这些Matcher：

- error.type
  匹配Error的type（name），对应的是JSON中的`metadata.type`，大小写敏感：

```glob
error.type:ZeroDivisionError -> zero-division
error.type:ConnectionError -> connection-error
```

- error.value
  匹配Error的value（message），对应的是JSON中的`metadata.value`，允许使用通配符，大小写不敏感：

```glob
error.value:"connection error (code: *)" -> connection-error
error.value:"could not connect (*)" -> connection-error
```

- message
  匹配日志消息，对应的是JSON中的`message`字段，允许使用通配符，大小写不敏感：

```glob
message:"system encountered a fatal problem: *" -> fatal-log
```

- logger
  匹配当前的logger的名称，对应的是JSON中的`logger`字段，允许使用通配符，大小写敏感：

```glob
logger:"com.myapp.mypackage.*" -> mypackage-logger
```

- level
  匹配当前Event的日志级别，对应的是JSON中的`level`字段，允许使用通配符，大小写不敏感：

```glob
logger:"com.myapp.FooLogger" level:"error" -> mylogger-error
```

- tags.tag_name
  匹配某个tag，某个标签名，允许使用通配符。
  这里的tag_name，对应的是JSON中的`tags`字段中，每一项的名字。tags是一个数字，代表多个标签，每一项是一个标签，每个标签也是一个数字，数组有两个元素，第一个元素是标签名，即tag_name，第二个是标签值。例如下面这样：

```json
{  
  "tags": [
    [
      "device",
      "iPhone10,2"
    ],
    [
      "device.family",
      "iOS"
    ],
    [
      "os",
      "iOS 16.1.2"
    ],
    [
      "os.name",
      "iOS"
    ],
    [
      "environment",
      "dev"
    ],
    [
      "release",
      "dev-v3.24"
    ],
    [
      "dist",
      "3.24.1023"
    ],
    // ......
  ]
}
```

```glob
tags.release:"dev-v3.x" -> dev-v3-error
```

- stack.abs_path
  匹配调用栈帧的绝对路径，对应的是每一帧中的`abs_path`字段，允许使用通配符，且大小写不敏感：

```glob
stack.abs_path:"**/my-utils/*.js" -> my-utils, {{ error.type }}
```

- stack.module
  匹配调用栈帧的模块名，对应的是每一帧中的`module`字段，允许使用通配符，大小写敏感：

```glob
stack.module:"*/my-utils/*" -> my-utils, {{ error.type }}
```

- stack.function
  匹配调用栈帧的方法名，对应的是每一帧中的`function`字段，大小写敏感：

```glob
stack.function:"my_assertion_failed" -> my-assertion-failed
```

- stack.package
  匹配当前帧所在的`package`，允许使用通配符：

```glob
stack.package:"**/libcurl.dylib" -> libcurl
stack.package:"**/libcurl.so" -> libcurl
```

- family
  通常用来缩小匹配范围，且通常和其他Matcher一起使用，目前包含以下值：
  - javascript，任何来自于JavaScript的Event
  - native，任何来自于Native的Event
  - other，其他任何Event

```glob
family:native !stack.module:"myproject::*" -> not-from-my-project
```

- app
  匹配当前帧是否是在app内，通常和其他Matcher一起使用，包含yes和no两个值，对应的是每一帧中的in_app字段：

```glob
app:yes stack.function:"assert" -> assert
```

更多关于Matchers的信息，详见[Matchers](https://docs.sentry.io/product/data-management-settings/event-grouping/fingerprint-rules/#matchers)。

#### 4.2 Variables

在一条Fingerprint Rule的右侧，就是Variables，这里其实不只可以是变量，也可以是一些写死的常量。

对于变量来说，它们和Matchers的名字一样，并且会自动把变量对应的原始的值填入，用于生成Fingerprint。

例如：

```glob
stack.function:"evaluate_script" -> script-evaluation, {{ error.type }}
```

这条规则会匹配调用栈中方法名为`evaluate_script`的Error，并且会把常量`script-evaluation`和当前Error的type（name）作为一部分，一起生成Fingerprint。

例如，`["script-evaluation", "ReferenceError"]`

或者，`["script-evaluation", "TypeError"]`

其他的变量和Matchers的名字一样，都是使用`{{ }}`包裹起来的，详见[Variables](https://docs.sentry.io/product/data-management-settings/event-grouping/fingerprint-rules/#variables)。

#### 4.3 自定义标题

在设置Fingerprint Rules时，我们往往是想要按照自己的规则对Event进行分组，但是Event通常都是使用type和value来作为标题展示在Sentry中的，如果只改了Fingerprint Rules，那么原始的Event标题可能不那么友好，或者具有一定的误导性。

这个时候，我们可以在添加Fingerprint Rules的时候，额外添加title字段，即可设置这个分组的标题。比如：

```glob
logger:my.package.* level:error -> error-logger, {{ logger }} title="Error from Logger {{ logger }}"
```

**自定义标题前：**

![自定义标题前](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/issue_before_rename.png)

**自定义标题后：**

![自定义标题后](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/issue_after_rename.png)

在设置了自定义标题后，就可以在Error的原始JSON数据中看到title发生了变化：

![自定义标题后的JSON](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/title_changed_in_json.png)

> **🔔 注意：**
>
> 只有比较新的版本（比如Sentry 23.x）才支持设置自定义的title在旧版本的Sentry中（比如Sentry 20.x），上面的写法会让Sentry把后面的`title="Error from Logger {{ logger }}"`认为是Fingerprint的一部分。
>
> 具体是哪个版本开始支持的我没在网上查到，如果你私有部署的Sentry版本发现不支持，可以尝试升级一下版本。

#### 4.4 怎么确定有没有匹配上自定义的Fingerprint Rules?

在添加了自定义的Fingerprint Rules之后，我们如何确定某个Event有没有命中呢？

其实我们直接查看对于的JSON数据即可，如果匹配上的话，会看到下图这样：

![匹配上的JSON](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/finger_print_in_json.png)

我们可以在_fingerprint_info中看到当前Event的各种信息

- `client_fingerprint`，对应的是这个Event的SDK Fingerprint（下面会讲到，当前Event没有设置SDK Fingerprint，所以为default）
- `matched_rule`，对应的是Fingerprint Rules，比如这里显示当前Event命中的Matchers是哪个，以及当前Matchers设置的Fingerprint，还有我们自定义的title

同时下面的`fingerprint`字段，也展示了当前Event最终的Fingerprint。

如果是旧版Sentry的话，这里就没有`_fingerprint_info`这个字段了，同时会把我们设置的title认为是Fingerprint的一部分，会是下面这样：

![旧版Sentry的JSON](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/old_version_custom_title_in_json.png)

### 5. SDK Fingerprint

如果上面说的Fingerprint Rules，不能满足我们的需要，那么我们可以使用SDK Fingerprint来更灵活地生成Fingerprint。

> 从上面的Fingerprint Rules文档可以看到，Fingerprint Rules只有少部分Matchers支持设置通配符，所以可能没那么灵活。

如果从上面的官方文档[Issue Grouping | Sentry Documentation](https://docs.sentry.io/product/data-management-settings/event-grouping/)中看，会发现文档里没有单独SDK Fingerprint的文档。

这是因为SDK Fingerprint是针对不同的Sentry SDK的，不同的项目会使用不同的Sentry SDK。每个SDK中设置Fingerprint的方式都不一样，甚至可能部分SDK不支持设置Fingerprint。所以要针对不同的平台，查看各自平台的SDK文档，这里以JavaScript Sentry SDK为例[SDK Fingerprinting for Browser JavaScript](https://docs.sentry.io/platforms/javascript/usage/sdk-fingerprinting/)。

*更多平台请看这里：[Platforms](https://docs.sentry.io/platforms/)*

官网文档提供了比较友好的三个例子：

#### 5.1 基础示例

单独处理某个上报Event的Fingerprint

```javascript
function makeRequest(method, path, options) {
  return fetch(method, path, options).catch(function(err) {
    Sentry.withScope(function(scope) {
      // group errors together based on their request and response
      scope.setFingerprint([method, path, String(err.statusCode)]);
      Sentry.captureException(err);
    });
  });
}
```

我们可以使用变量替换，把一些Fingerprint Rules支持的变量填入，作为我们设置的Fingerprint的一部分，比如`{{ default }}`，`{{ stack.abs_path }}`，`{{ error.type }}`等，详见上面提到的Fingerprint Rules变量。

#### 5.2 更细粒度地控制分组

在原有的Fingerprint后拼接上自定义的一些字段，可以达到比默认的规则更细粒度的控制。

比如下面例子，进一步拆分Sentry创建的默认分组（由`{{ default }}`表示），同时考虑错误对象的一些属性：

```javascript
class MyRPCError extends Error {
  constructor(message, functionName, errorCode) {
    super(message);

    // The name of the RPC function that was called (e.g. "getAllBlogArticles")
    this.functionName = functionName;

    // For example a HTTP status code returned by the server.
    this.errorCode = errorCode;
  }
}

Sentry.init({
  // ...
  beforeSend: function (event, hint) {
    const exception = hint.originalException;

    if (exception instanceof MyRPCError) {
      event.fingerprint = [
        "{{ default }}",
        String(exception.functionName),
        String(exception.errorCode),
      ];
    }

    return event;
  },
});
```

#### 5.3 完全重写Fingerprint

还可以直接整个重写Fingerprint

```javascript
class DatabaseConnectionError extends Error {}

Sentry.init({
  // ...
  beforeSend: function (event, hint) {
    const exception = hint.originalException;

    if (exception instanceof DatabaseConnectionError) {
      event.fingerprint = ["database-connection-error"];
    }

    return event;
  },
});
```

#### 5.4 和Fingerprint Rules怎么划分职责？

Fingerprint Rules和SDK Fingerprint都可以实现相同的功能，那么我们在设置自定义Fingerprint的时候，要怎么取舍，或者说什么时候用Fingerprint Rules，什么时候用SDK Fingerprint？

**Fingerprint Rules：**

- 优势：
  - 可以随时修改规则，不需要进行代码的变更
  - 可以同时在线上所有版本生效
- 劣势：
  - 没有SDK Fingerprint灵活，有些处理不了，比如error.type不支持通配符匹配

**SDK Fingerprint：**

- 优势：
  - 灵活，可以用JS很方便地处理或者自定义Fingerprint
- 劣势：
  - 需要修改代码
  - 分组规则和代码版本耦合，如果应用需要用户手动升级的话，那么旧版本应用内的Sentry上报没办法处理

从上面的优劣对比来看，可以看到**Fingerprint Rules和SDK Fingerprint是优劣互补的，一方的优势恰好是另一方的劣势**。

对比下来，我们在项目中使用的时候，建议**如果可以使用Fingerprint Rules实现的，都用Fingerprint Rules，只有在Fingerprint Rules无法满足的情况下，再用考虑使用SDK Fingerprint**。

### 6. Filter

上面说了这么多关于Issue分组的，那么对于一些我完全不想要的上报，有没有办法完全不分组，直接过滤掉呢？

也是有的，可以在Sentry平台上，直接设置一些Filter过滤器来过滤，而不需要我们手动在使用SDK的地方修改。

具体设置位置为：**Settings > Project > [Your Project] > Processing > Inbound Filters**。

过滤器分为内置的一些过滤器，和自定义的过滤器。

#### 6.1 内置过滤器

Sentry平台内置了一些可以直接启用的过滤器，这些过滤器包括：

- 浏览器拓展插件的error
- 来自于localhost的event
- 已知的旧版浏览器错误，比如IE的
- 已知的网络爬虫错误
- React hydrate的报错（和React服务端渲染有关的错误）
- ......

这些过滤器可能和不同版本的Sentry有关，比较旧的版本中，可能会缺少一些过滤器。

![内置过滤器](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/built_in_filters.png)

#### 6.2 自定义过滤器

可以创建自定义过滤器，目前支持以下三种，以下三种在匹配时，都是大小写不敏感的。

![自定义过滤器](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/custom_filters.png)

##### a. 特定IP地址

可以设置IP地址，过滤特定IP的错误上报，比如`127.0.0.1`。

##### b. 特定发布版本

- 发布版本，指的是在Sentry.init的时候，传入的release字段。
- 可以使用通配符，比如production-v3.24.*
- 如果某个Event不包含release字段，那么这个Event不会被过滤

```javascript
Sentry.init({
  dsn: 'xxx',
  release: `${env}-${version}`,
  ...
});
```

如果不确定自己项目上报后最终的`release`字段是什么，可以直接查看任意一个Error Event的原始JSON数据中的`release`字段（前提是init时传入了这个字段或者Event数据中有这个字段）

##### c. Error Message

- 可以设置多个匹配项，每行一个。只要任意一个匹配项匹配成功，那么就会过滤这一条上报
- 对于Error类型的Event，会根据设置的匹配项，对格式为`{exception.type}: {exception.value}`的整个错误描述进行匹配。
  但是不建议直接匹配整个描述，比如把冒号也加在里面，一般都是通过通配符来进行匹配。比如`*ConnectionError*`
- Transaction类型的Event，不会被过滤

在设置完之后，可以检查下Issue的原始JSON数据，设置的过滤器会根据JSON里的`title`字段进行匹配，可以检查下是否有问题。

在设置好过滤器之后，我们就可以看到有多少Event被过滤掉了：

![过滤掉的Event](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/sentry_issue_grouping/filterd_issues.png)

## 五、治理

到这里我们已经搞明白Sentry对Issue分组的原理了，也知道了怎么自定义分组。那我们是使用Stack Trace Rules还是Fingerprint Rules来处理呢？

使用Stack Trace Rules本质上还是根据调用栈来进行分组，但是这就需要我们必须上传Source Maps。

在上传了Source Maps的情况下，可以通过设置调用栈Stack Trace Rules来[裁切调用栈](https://docs.sentry.io/product/data-management-settings/event-grouping/stack-trace-rules/#cut-stack-traces)，或者限制Sentry在生成调用栈Fingerprint的时候需要考虑的[top帧数量](https://docs.sentry.io/product/data-management-settings/event-grouping/stack-trace-rules/#stack-trace-frame-limits)。

考虑到目前我们的项目因为某种原因，还不能上传Source Maps，同时代码每个版本变化可能会导致同样的问题的调用堆栈信息不同。基于我们的需求来看，完全自定义的Fingerprint Rules更符合我们的情况。

所以我们的项目会做如下处理：

- 在Sentry平台上设置Fingerprint Rules，处理绝大部分可以处理的Error
- 少部分Fingerprint Rules无法处理的Error（比如error.type不支持通配符），通过SDK Fingerprint，在代码中Sentry.init的时候，增加`beforeSend`进行处理
- 一些不需要关注的Error，设置Inbound Filters直接过滤

## 六、小结

一番调研下来，通过Fingerprint Rules，Stack Trace Rules，SDK Fingerprint，以及Inbound Filters，我们把项目的Issue进行了自定义分组，更方便我们排查问题，分析处理。

Sentry是一个简单易上手的监控平台，但是Sentry上也有许多十分复杂的配置项，这篇文章只是Sentry文档的一小部分，有哪里不正确的，还请多多指正。
