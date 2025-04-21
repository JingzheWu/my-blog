---
title: 一起从零实现React SSR服务端渲染
date: 2023-12-21 17:25:00
tags:
---

## 前言

最近 Next.js 14 发布了，很多地方都在讨论，虽然之前也有用过 Next.js，也看过一些关于 SSR 的文章，了解了一些 SSR 的原理，但是一直没有动手实现过。这次正好趁着这个机会，从零开始手动实现一个 React SSR 服务端渲染的项目，来加深一下对 SSR 的理解。

## 一、 什么是SSR

SSR，即Server Side Render，服务端渲染。和服务端渲染相对的，就是CSR，Client Side Render，客户端渲染。

从字面意思上就有看出，这两种渲染方式的差别就在于页面渲染的时机：

- 服务端渲染是页面在服务端的时候就渲染完成了
- 而客户端渲染是页面在客户端（浏览器或者WebView之类的）进行渲染，服务端只会返回空页面（下面例子中会讲到）

## 二、 为什么需要SSR

在讨论为什么需要SSR之前，我们先来看看常见的CSR，比如下面这个很简单的React渲染的页面：

![page_overview.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/page_overview.png)

页面包含一个count计数，点击“Increment”、“Decrement”和“Reset”按钮，分别可以增加计数，减小计数以及重设计数。

现在我们打开DevTool，看看访问这个地址的时候服务端返回的内容：

![csr_html_content.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/csr_html_content.png)

从DevTool可以看到，服务端一共返回了两个文件，一个HTML一个JS。

**我们发现HTML里没有任何页面上的元素，是空的**，只有一个root节点和一个main.js脚本。而所有的这些count计数以及下面这些按钮，都是在HTML页面以及main.js下载完成后，浏览器通过执行main.js来进行渲染生成的，这也就是所谓的客户端渲染。

通过这个我们可以发现，CSR的应用有两个比较明显的问题：

### 1. CSR应用十分不利于SEO

SEO，也就是Search Engine Optimization，搜索引擎优化。CSR应用从服务端返回的HTML是一个空的页面，页面内容元素完全依赖JS代码来生成，导致搜索引擎爬虫在抓取和解析网页时，无法获取到完整的网页内容，从而不利于搜索引擎优化搜索结果和搜索排名。

> 题外话：现在的一些搜索引擎（比如Google）已经可以解析CSR应用里的JS，所以CSR目前对SEO的影响可能没有以前那么大了，但是如果SEO对你来说很重要，那么最好还是做一些SSR服务端渲染

### 2. 首屏加载时间可能较长

由于CSR应用页面里所有的内容，都是通过JS动态生成的，那么在访问页面的时候，除了下载HTML外，还需要额外下载JS脚本才可以展示出页面。

衡量首屏加载性能的指标有很多，我们这里用常用的[FCP（First Contentful Paint）](https://developer.chrome.com/docs/lighthouse/performance/first-contentful-paint?utm_source=devtools)，即“首次内容渲染”时间来看下这个页面的表现。由于我们这个页面太过简单，而且是在本地`127.0.0.1`启动的服务，所以直接感受可能不明显，我们可以在DevTool里设置网络状态，改成“低速3G”来模拟：

![csr_network_panel.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/csr_network_panel.png)

而FCP除了可以用`performance` API获取到之外，也可以直接在Chrome DevTool的“性能”面板，通过点击面板里的“重新加载”按钮录制得到：

![csr_perf_panel.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/csr_perf_panel.png)

从上面的网络瀑布和性能面板可以看到，在“低速3G”的网络状态下，页面在获取到2.01s获取到HTML后，并没有渲染任何内容，而是在又等了4.76s等到JS下载完成之后，才渲染出内容，页面的FCP总计是6822.2ms。

也就是说，在2.01s的时候，HTML已经下载好的情况下，用户还是无法看到内容，额外等待了4.76s等待JS的下载，首屏加载时间较长。

下面我们就来尝试把上面这个项目改成支持服务端渲染，来改善这两个CSR的弊端。

## 三、 准备好一个项目

首先我们先准备好上面这个项目的代码，这是项目的GitHub地址，大家可自取：[react-no-ssr-demo](https://github.com/JingzheWu/react-no-ssr-demo)

项目的目录结构长这样：

```plaintext
.
├── src
│   ├── components
│   │   └── Button
│   │       ├── index.module.css
│   │       └── index.tsx
│   ├── Home.module.css
│   ├── Home.tsx
│   ├── index.html
│   └── index.tsx
├── webpack
│   └── client.config.js
├── .babelrc
├── .eslintignore
├── .eslintrc.json
├── .gitignore
├── .prettierignore
├── .prettierrc.json
├── package-lock.json
├── package.json
├── README.md
├── tsconfig.json
└── types.d.ts
```

关键的几个文件内容如下：

1. `src/components/Button/index.tsx`

```typescript
import React from 'react';
import classNames from 'classnames';
import styles from './index.module.css';

export interface IButtonProps {
  text: string;
  className?: string;
  onClick?: () => void;
}

export const Button: React.FC<IButtonProps> = ({
  text,
  className,
  onClick,
}) => {
  return (
    <button onClick={onClick} className={classNames(styles.btn, className)}>
      {text}
    </button>
  );
};
```

2. `src/Home.tsx`

```typescript
import React, { useState } from 'react';
import { Button } from '@/components/Button';
import styles from './Home.module.css';

export const Home: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p style={{ color: 'red' }}>count: {count}</p>
      <div className={styles.btnWrapper}>
        <Button
          text="Increment"
          className={styles.btn}
          onClick={() => setCount(prev => prev + 1)}
        />
        <Button
          text="Decrement"
          className={styles.btn}
          onClick={() => setCount(prev => prev - 1)}
        />
        <Button
          text="Reset"
          className={styles.btn}
          onClick={() => setCount(0)}
        />
      </div>
    </div>
  );
};
```

3. `src/index.tsx`

```typescript
import React from 'react';
import ReactDOM from 'react-dom';
import { Home } from './Home';

ReactDOM.render(<Home />, document.getElementById('root'));
```

4. `webpack/client.config.js`

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');

const rootDir = path.resolve(__dirname, '..');

module.exports = {
  entry: './src/index.tsx',
  output: {
    path: path.resolve(rootDir, 'dist/client'),
    filename: 'main.js',
  },
  resolve: {
    modules: ['node_modules', path.resolve(rootDir, 'src')],
    extensions: ['.ts', '.js', '.tsx'],
  },
  module: {
    rules: [
      {
        test: /\.(tsx?|jsx?)$/,
        use: 'babel-loader',
        exclude: '/node_modules/',
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
      inject: 'body',
      scriptLoading: 'defer',
    }),
    new CleanWebpackPlugin(),
    new ForkTsCheckerWebpackPlugin({}),
  ],
  performance: {
    hints: false,
    maxEntrypointSize: 512000,
    maxAssetSize: 512000,
  },
};
```

5. `package.json`

```json
{
  "scripts": {
    "dev": "webpack --config webpack/client.config.js --mode development --watch --stats verbose",
    "build": "webpack build --config webpack/client.config.js --mode production --stats verbose"
  }
}
```

除此之前，其他一些都是代码规范化相关的配置，比如eslint、prettier之类的，不是这次重点讨论的范围，感兴趣的可以看我之前这篇文章[《前端代码规范化配置最佳实践 - 掘金》](https://juejin.cn/post/7314365567376162853)。

通过在本地执行`npm run dev`或者`npm run build`，就可以编译项目，这里我没有使用webpack dev server，而是起了一个Nginx服务，简单改了一下Nginx配置：

```nginx
server {
  listen 3006;
  server_name localhost;

  location / {
    root   /your-project-location/react-no-ssr-demo/dist/client;
    index  index.html index.htm;
  }
}
```

这样直接在浏览器访问`127.0.0.1:3006`或者`localhost:3006`就可以了。

## 四、 使用服务端渲染

现在让我们开始给项目添加服务端渲染，既然要服务端渲染，就必须在服务端使用React的[renderToString](https://react.docschina.org/reference/react-dom/server/renderToString)来渲染好HTML再返回给客户端，既然要求在服务端跑JS代码，那么服务端就必须要引入Node了，也就是说做服务端渲染，必须要有一个Node做中间层（Next.js的[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)不算是严格意义上的服务端渲染，不在讨论范围内）。

### 1. 添加Node服务

现在让我们先加一个基础的Node服务：

```bash
npm install --save express
npm install --save-dev @types/express
```

添加一个`src/server.tsx`：

```typescript
import express from 'express';
import React from 'react';
import ReactDOMServer from 'react-dom/server';
import { Home } from './Home';

const app = express();

app.get('/', (req, res) => {
  const app = ReactDOMServer.renderToString(<Home />);
  const html = `
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8">
        <title>React SSR</title>
      </head>
      <body>
        <div id="root">${app}</div>
      </body>
    </html>
  `;
  res.send(html);
});

const PORT = process.env.PORT || 3007;
app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`);
  console.log(`http://localhost:${PORT}`);
  console.log(`http://127.0.0.1:${PORT}`);
});
```

这里我们换了一个`3007`端口，为了和刚刚的CSR项目的`3006`端口区分开来（另外使用了Node服务之后，也就不需要Nginx了）。

从上面的代码可以看到，`src/server.tsx`中，先创建了一个express服务，然后监听了`3007`端口，在访问`127.0.0.1:3007`或者`localhost:3007`的时候，服务端调用ReactDOMServer的`renderToString`方法，将我们的`Home`组件渲染为了HTML字符串，并且拼接到了一个HTML模板中，返回给了客户端。

### 2. 添加`src/server.tsx`的编译配置

由于`src/server.tsx`使用了TS和JSX的语法，那么这个文件也需要使用webpack和babel进行编译，让我们添加一下这个server文件的编译配置，`webpack/server.config.js`：

```js
const path = require('path');

const rootDir = path.resolve(__dirname, '..');

module.exports = {
  target: 'node',
  entry: './src/server.tsx',
  output: {
    path: path.resolve(rootDir, 'dist'),
    filename: 'server.js',
  },
  module: {
    rules: [
      {
        test: /\.(tsx?|jsx?)$/,
        use: 'babel-loader',
        exclude: '/node_modules/',
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
};
```

然后在`package.json`中添加一下编译服务端的scripts命令：

```json
{
  "scripts": {
    "start": "node dist/server.js",
    "dev:server": "webpack --config webpack/server.config.js --mode development --watch --stats verbose",
    "build:server": "webpack build --config webpack/server.config.js --mode production --stats verbose"
  }
}
```

执行`npm run dev:server`以及`npm start`之后，打开`127.0.0.1:3007`或者`localhost:3007`看下：

![ssr_no_js.gif](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_no_js.gif)

果然，这次服务端返回的HTML不再是空的了，页面上的元素直接就可以在HTML中看到。

但是这里奇怪的是，页面的样式没有了，点击这些按钮也不再生效了，同时从网络请求里看到，服务端也只返回了一个HTML，没有JS，这是为什么呢？

## 五、 同构 & hydrate

其实，这是因为我们上面实现的SSR还不是一个完整的React SSR项目。`renderToString`虽然可以在服务端把组件渲染为HTML，但是却无法实现事件监听器的挂载或者绑定（毕竟事件绑定是要绑定到浏览器上真实的DOM上，而不是HTML字符串上），所以在`renderToString`的时候会把事件处理器给过滤掉。

### 1. 同构

那么为了实现完整的SSR，就需要引入“同构渲染”的概念了。这个词相信大家之前都或多或少听过，其实很简单，同构渲染就是同一份代码，既在服务端运行（SSR），又在客户端运行（CSR）。

最开始我们提到的那个CSR代码，只是在客户端运行，后来加上的服务端渲染的能力，只是在服务端运行（客户端只是接收了一个HTML，并没有运行什么JS代码）。现在需要将两者结合起来，接下来让我们开始改造一下：

首先，`src/server.tsx`中，我们不再直接返回一个模板HTML，而是在上面CSR项目编译出来的HTML中直接加上服务端渲染的内容，同时在服务端提供静态资源访问服务：

```typescript
import fs from 'fs';
import path from 'path';
import express from 'express';
import React from 'react';
import ReactDOMServer from 'react-dom/server';
import { Home } from './Home';

const clientDistDir = path.resolve(__dirname, '../dist/client');
const htmlPath = path.resolve(clientDistDir, 'index.html');

const app = express();

app.get('/', (req, res) => {
  // 读取 dist/client/index.html 文件
  const html = fs.readFileSync(htmlPath, 'utf-8');
  const app = ReactDOMServer.renderToString(<Home />);
  // 将渲染后的 React HTML 插入到 div#root 中
  const finalHtml = html.replace(
    '<div id="root"></div>',
    `<div id="root">${app}</div>`
  );
  res.send(finalHtml);
});

// 提供静态资源访问服务
app.use(express.static(clientDistDir));

const PORT = process.env.PORT || 3007;
app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`);
  console.log(`http://localhost:${PORT}`);
  console.log(`http://127.0.0.1:${PORT}`);
});
```

直接在命令行执行：

```bash
npm run dev
npm run dev:server
npm start
```

打开`127.0.0.1:3007`或者`localhost:3007`看下：

![ssr_render.gif](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_render.gif)

看起来好像OK了，既有服务端渲染（返回的HTML不为空，直接就有页面上的元素），又有客户端渲染（事件绑定成功，有页面交互）。但是如果这个时候你查看一下控制台的话，会发现会有一个Waring：

![ssr_render_warn.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_render_warn.png)

说是调用`ReactDOM.render()`去渲染（水合，或者说注水）一个服务端渲染的页面的行为会在React 18停止支持。

### 2. hydrate

其实这个从[React官网](https://react.dev/reference/react-dom/server/renderToString#reference)也可以看到，在服务端使用`renderToString`外进行服务端渲染后，还需要在客户端使用`hydrate`（或者`hydrateRoot`，后者是React 18中的写法），来完成事件绑定和页面的交互性逻辑。

来改下代码，在`src/index.tsx`中，改为如下内容：

```typescript
import React from 'react';
import ReactDOM from 'react-dom';
import { Home } from './Home';

ReactDOM.hydrate(<Home />, document.getElementById('root'));
```

同时为了方便调试，安装一下两个依赖：

```bash
npm install --save nodemon
npm install --save-dev npm-run-all
```

修改`package.json`：

```json
{
  "scripts": {
    "start": "nodemon --inspect dist/server.js",
    "dev": "npm-run-all --parallel dev:*",
    "dev:client": "webpack --config webpack/client.config.js --mode development --watch --stats verbose",
    "dev:server": "webpack --config webpack/server.config.js --mode development --watch --stats verbose",
    "build": "npm-run-all build:*",
    "build:client": "webpack build --config webpack/client.config.js --mode production --stats verbose",
    "build:server": "webpack build --config webpack/server.config.js --mode production --stats verbose"
  }
}
```

其中`nodemon`用于监听`dist/server.js`的变化，一旦修改了`src/server.tsx` webpack会重新编译，生成新的`dist/server.js`，这个时候nodemon会重新运行新的`dist/server.js`。

`npm-run-all`则用于同时运行多个npm命令。

这个时候再运行：

```bash
npm run dev
npm start
```

访问`127.0.0.1:3007`或者`localhost:3007`，发现已经没有Waring了：

![ssr_hydrate_no_warn.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_hydrate_no_warn.png)

### 3. hydrate和render的区别

`render()`和`hydrate()`在大部分情况下的行为是相似的，这两个都会将React元素渲染到指定的DOM节点中，但是在处理服务端渲染返回的HTML是有一些区别。

服务端渲染的时候，服务端会渲染React元素并且生成一个HTML字符串返回给客户端（也就是浏览器），之后客户端会用这个HTML来生成DOM。在同构渲染的时候，客户端还会重新执行一遍JS代码，重新生成一个React组件树和相应的DOM节点。而`render()`和`hydrate()`的区别就在这里。

`render()`会直接创建一个新的React组件数和相应的DOM节点，而`hydrate()`则是在生成的时候，会判断这个节点是否已经在服务端渲染好，**会尽可能地保留现有的DOM，只更新必要的部分**。

这也就是React官网所说的：

> Call hydrate in React 17 and below to “attach” React to existing HTML that was already rendered by React in a server environment.
>
> React will attach to the HTML that exists inside the domNode, and take over managing the DOM inside it.

在React 17及以下版本中调用`hydrate`，可以将React“附加”到在服务器环境中已经由React渲染的现有HTML上。

React将会附加到`domNode`内部现有的HTML，并接管有关的DOM的管理。

## 六、 再来看下CSR的两个痛点

到这里就算是完成了一个最基本的React服务端渲染，现在我们回头来看一下，是否解决了上面CSR项目的两个痛点。

### 1. SEO

首先是之前SEO不友好的问题。

在做了SSR渲染后，从服务端返回的HTML里就已经包含了页面上的元素，搜索引擎爬虫在抓取和解析网页时，可以获取到完整的网页内容，显然SSR渲染可以解决这个问题（当前想要更好的SEO效果，还有其他可以优化的地方，不过这些就和CSR/SSR无关了）。

### 2. FCP

其次，我们来看一下首屏的加载时间，还是通过设置DevTool里设置网络状态，改成“低速3G”来看一下FCP：

![ssr_network_panel.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_network_panel.png)

![ssr_perf_panel.png](https://youfindme-1254464911.cos.ap-hongkong.myqcloud.com/blog/react_ssr_from_scratch/ssr_perf_panel.png)

从上面可以看到，虽然网络面板内的HTML和JS整体的加载时间和之前几乎一样（都是6.8s左右），但是从性能面板里可以看到，页面的FCP是2043.2ms，比之前的6822.2ms少了将近70%。

从时间轴的截图上也可以发现，页面在HTML下载成功之后（2.02s），就立刻可以看到页面内容，虽然页面的交互还是需要等到6.8s JS下载完成，但是从用户体验上来讲，缩短页面的白屏时间，用户可以更快的看到页面内容，对于用户体验是一个很大的提升。

也就是说，SSR确实解决了CSR的两个痛点。

## 七、 服务端渲染一些主要注意的事情

下面是一些做服务端渲染时需要注意的点：

### 1. React的生命周期和一些Hooks

React的一些生命周期函数，比如类组件的`componentDidMount`，`componentDidUpdate`和`componentWillUnmount`，以及函数组件的`useEffect`和`useLayoutEffect`，都不会在服务端渲染的时候执行。

### 2. 浏览器专属的API

浏览器专属的API，比如`window`，`document`，`localStorage`等，都不能在服务器端运行，需要判断只有在当前环境是客户端才可以执行：

```javascript
if (typeof window !== 'undefined') {
  // 下面的代码只会在浏览器环境下执行
  window.localStorage.setItem('key', 'value');
}
```

或者

```javascript
useEffect(() => {
  // 下面的代码只会在浏览器环境下执行
  window.localStorage.setItem('key', 'value');
}, []);
```

### 3. 事件处理函数

如上面提到的那样，服务端渲染的时候，不会执行事件处理函数，也不会触发任何事件，需要在客户端处理。

### 4. 服务端渲染和客户端渲染时的差异

在进行同构渲染的时候，**请务必保证**客户端渲染出来的内容和服务端渲染的内容完全相同。如果客户端和服务端渲染出来的内容不一致，React会尝试对不一致的地方进行修复，而这些修复是[非常耗时的](https://react.dev/reference/react-dom/hydrate#caveats)。如果差异过大甚至会重新渲染整个应用（类似于`ReactDOM.render`）。

所以应尽量避免客户端渲染出来的内容和服务端渲染出来的内容不一致。

## 八、 小结

通过上面的内容，我们从零手动完成了一个React服务端渲染的Demo项目，这只是一个最基础的项目，还有更多的比如React路由服务端渲染、服务端渲染时的数据脱水和注水等等，都需要添加更加复杂的配置，有时间了再单独写一篇聊一下。

这里附上文章里提到的两个Demo项目地址：

- CSR：[react-no-ssr-demo](https://github.com/JingzheWu/react-no-ssr-demo)
- SSR：[react-ssr-demo](https://github.com/JingzheWu/react-ssr-demo)

## 参考资料

- [https://react.dev](https://react.dev)
- [从头开始，彻底理解服务端渲染原理(8千字汇总长文) - 掘金](https://juejin.cn/post/6844903881390964744)
- [http://www.ayqy.net/blog/react-ssr-under-the-hood/#articleHeader0](http://www.ayqy.net/blog/react-ssr-under-the-hood/#articleHeader0)
- [什么是前端的同构渲染？](https://www.zhihu.com/question/325952676)
- [https://medium.com/%E6%89%8B%E5%AF%AB%E7%AD%86%E8%A8%98/server-side-rendering-ssr-in-reactjs-part1-d2a11890abfc](https://medium.com/%E6%89%8B%E5%AF%AB%E7%AD%86%E8%A8%98/server-side-rendering-ssr-in-reactjs-part1-d2a11890abfc)
