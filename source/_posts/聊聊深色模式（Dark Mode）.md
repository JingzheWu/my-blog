---
title: 聊聊深色模式（Dark Mode）
date: 2023-03-13 10:29:00
tags:
---

## 什么是深色模式

深色模式（Dark Mode），或者叫暗色模式，黑夜模式，是和日常使用的浅色（亮色）模式（Light Mode）相对应的一种UI主题。

深色模式最早来源于人机交互领域的研究和实践，从2018年左右开始，Apple推出了**iOS 13**，其中包含了系统级别的深色模式，可以将整个系统的界面切换为暗色调。

Google也在**Android 10**中推出了类似的深色模式功能，使深色模式得到了更广泛的应用和推广。

![iOS官网的深色模式示例](https://static.youfindme.cn/blog/dark_mode/dark_mode_example.png)

它不是简单的把背景变为黑色，文字变为白色，而是一整套的配色主题，这种模式相比浅色模式更加柔和，可以减少亮度对用户眼睛造成的刺激和疲劳。

随着越来越多的应用开始支持深色模式，作为开发也理应多了解下深色模式。

## 首先，怎么打开深色模式

在说怎么实现之前，先来说说我们要怎么打开深色模式，一般来说只需要在系统调节亮度的地方就可以调节深色模式，具体我们可以看各个系统的官方网站即可：
如何打开深色模式

- [在 iPhone 和 iPad 上使用深色模式 - 官方 Apple 支持 (中国)](https://support.apple.com/zh-cn/HT210332)
- [在 Mac 上使用深色模式 - 官方 Apple 支持 (中国)](https://support.apple.com/zh-cn/HT208976)
- [在 Android 设备上更改为深色模式或颜色模式 - Android帮助](https://support.google.com/android/answer/9730472?hl=zh-Hans)
- [在 Windows 中更改颜色 - Microsoft 支持](https://support.microsoft.com/zh-cn/windows/%E5%9C%A8-windows-%E4%B8%AD%E6%9B%B4%E6%94%B9%E9%A2%9C%E8%89%B2-d26ef4d6-819a-581c-1581-493cfcc005fe)

但是在开发调试调试时，不断切换深色模式可能比较麻烦，这时浏览器就提供了一种模拟系统深色模式的方法，可以让当前的Web页面临时变为深色模式，以Chrome为例：
浏览器模拟深色/浅色模式

1. 打开Chrome DevTools
2. `Command`+`Shift`+`P`
3. 输入dark或者light
4. 打开深色或者浅色模式![打开深色模式](https://static.youfindme.cn/blog/dark_mode/open_dark_mode_in_devtool.png)
   ![打开浅色模式](https://static.youfindme.cn/blog/dark_mode/open_light_mode_in_devtool.png)

不过要注意的是，浏览器DevTools里开启深色模式，在关闭开发者工具后就会失效。

## 自动适配 - 声明页面支持深色模式

其实，在支持深色模式的浏览器中，有一套默认的深色模式，只需要我们在应用中声明，即可自动适配深色模式，声明有两种方式：

### 1. 添加`color-scheme`的`meta`标签

在HTML的`head`标签中增加`color-scheme`的`meta`标签，如下所示：

```html
<!--
	The page supports both dark and light color schemes,
	and the page author prefers light.
-->
<meta name="color-scheme" content="light dark">
```

通过上述声明，告诉浏览器这个页面支持深色模式和浅色模式，并且页面更倾向于浅色模式。在声明了这个之后，当系统切换到深色模式时，浏览器将会把我们的页面自动切换到默认的深色模式配色，如下所示：

![左边浅色，右边是浏览器自动适配的深色](https://static.youfindme.cn/blog/dark_mode/left_light_right_auto_dark.png)

### 2. 在CSS里添加`color-scheme`属性

```css
/*
  The page supports both dark and light color schemes,
  and the page author prefers light.
*/
:root {
  color-scheme: light dark;
}
```

通过上面在`:root`元素上添加`color-scheme`属性，值为`light dark`，可以实现和`meta`标签一样的效果，同时这个属性不只可用于`:root`级别，也可用于单个元素级别，比`meta`标签更灵活。

但是提供`color-scheme`CSS属性需要首先下载CSS（如果通过`<link rel="stylesheet">`引用）并进行解析，使用`meta`可以更快地使用所需配色方案呈现页面背景。两者各有优劣吧。

## 自定义适配

### 1. 自动适配的问题

在上面说了我们可以通过一些标签或者CSS属性声明，来自动适配深色模式，但是从自动适配的结果来看，适配的并不理想：

![左边浅色，右边是浏览器自动适配的深色](https://static.youfindme.cn/blog/dark_mode/left_light_right_auto_dark.png)

- 首先是默认的黑色字体，到深色模式下变成了纯白色`#FFFFFF`，和黑色背景（虽然说不是纯黑）对比起来很扎眼，在一些设计相关的文章\[[1](https://36kr.com/p/1724109946881)]\[[2](https://www.woshipm.com/pd/4068702.html)]里提到，深色模式下避免使用纯黑和纯白，否则更容易使人眼睛👁疲劳，同时容易在页面滚动时出现拖影：

    ![滚动时出现拖影，图片来源「即刻」](https://static.youfindme.cn/blog/dark_mode/smearing_when_scrolling.png)

- 自动适配只能适配没有指定颜色和背景色的内容，比如上面的1、2、3级文字还有背景，没有显式设置`color`和`background-color`。

    对于设置了颜色和背景色（这种现象在开发中很常见吧）的内容，就无法自动适配，比如上面的7个色块的背景色，写死了颜色，但是色块上的文字没有设置颜色。最终在深色渲染下渲染出的效果就是，色块背景色没变，但是色块上的文字变成了白色，导致一些文字很难看清。

所以，最好还是自定义适配逻辑，除了解决上面的问题，还可以加一下其他的东西，比如加一些深浅色模式变化时的过渡动画等。

### 2. 如何自定义适配

自定义适配有两种方式，CSS媒体查询和通过JS监听主题模式

#### 1). CSS媒体查询

[prefers-color-scheme - CSS：层叠样式表 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-color-scheme)
我们可以通过在CSS中设置媒体查询`@media (prefers-color-scheme: dark)`，来设置深色模式下的自定义颜色。比如：

```css
.textLevel1 {
  color: #404040;
  margin-bottom: 0;
}
.textLevel2 {
  color: #808080;
  margin-bottom: 0;
}
.textLevel3 {
  color: #bfbfbf;
  margin-bottom: 0;
}

@media (prefers-color-scheme: dark) {
  .textLevel1 {
    color: #FFFFFF;
    opacity: 0.9;
  }
  .textLevel2 {
    color: #FFFFFF;
    opacity: 0.6;
  }
  .textLevel3 {
    color: #FFFFFF;
    opacity: 0.3;
  }
}
```

通过媒体查询设置元素在深色模式下的1、2、3级文字的颜色，在浅色模式下设置不同的颜色，在深色模式下，增加不透明度：

![左边的是自动适配的浅色深色，右边是自定义适配的浅色深色](https://static.youfindme.cn/blog/dark_mode/left_auto_right_manul.png)

对于`prefers-color-scheme`的兼容性也不必担心，主流浏览器基本都支持了：

![prefers-color-scheme](https://static.youfindme.cn/blog/dark_mode/prefers_color_scheme.png)

#### 2). JS监听主题颜色

[Window.matchMedia() - Web API 接口参考 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/matchMedia)

通过CSS媒体查询适配深色模式已经很方便了，完全不需要修改JS代码逻辑，那为什么还需要JS监听主题颜色呢？

因为通过CSS媒体查询虽然方便，但是只能跟随系统的主题颜色变化，假设用户想要类似于下面的自定义主题颜色，不跟随系统变化，或者业务上想做一些什么高级的操作，那么CSS媒体查询就无能为力了。

![custom_theme.gif](https://static.youfindme.cn/blog/dark_mode/custom_mode.gif)

在JS中，可以通过`window.matchMedia('(prefers-color-scheme: dark)').match`来判断当前的颜色主题：

```typescript
const darkModeMediaQuery = window.matchMedia('(prefers-color-scheme: dark)');

if (darkModeMediaQuery.matches) {
  // dark mode
} else {
  // light mode
}

darkModeMediaQuery.addEventListener('change', (event: MediaQueryListEvent) => {
  if (event.matches) {
    // dark mode
  } else {
    // light mode
  }
});
```

如果使用的是React，还可以写一个自定义hooks，来监听当前的颜色主题：

```typescript
export enum EColorScheme {
  LIGHT = 'light',
  DARK = 'dark',
}

export const useColorScheme = (): EColorScheme => {
  const [scheme, setScheme] = useState(
    darkModeMediaQuery.matches ? EColorScheme.DARK : EColorScheme.LIGHT
  );

  useEffect(() => {
    const listener = (event: MediaQueryListEvent) => {
      setScheme(event.matches ? EColorScheme.DARK : EColorScheme.LIGHT);
    };
    darkModeMediaQuery.addEventListener('change', listener);
    return () => {
      darkModeMediaQuery.removeEventListener('change', listener);
    };
  }, []);

  return scheme;
};
```

#### 一些题外话

我写完这个hooks之后，有一次偶然用Chat GPT让它写一段监听深色模式的代码，写成hooks，最后生成的代码和这个几乎一模一样，Chat GPT还是有点东西的：

```typescript
import { useState, useEffect } from 'react';

type ThemeType = 'light' | 'dark';

function useTheme(): ThemeType {
  const [theme, setTheme] = useState<ThemeType>(
    window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');

    const handleChange = () => setTheme(mediaQuery.matches ? 'dark' : 'light');

    mediaQuery.addEventListener('change', handleChange);

    return () => mediaQuery.removeEventListener('change', handleChange);
  }, []);

  return theme;
}

export default useTheme;
```

`window.matchMedia`的兼容性也挺好的：

![window.matchMedia](https://static.youfindme.cn/blog/dark_mode/window_match_media.png)

通过JS监听颜色主题变化之后，那可玩性就很多了，我们可以通过下面这些方式来适配深色模式：

- 动态添加类名覆盖样式

    通过判断深色模式来添加一个深色模式的类名，覆盖浅色模式样式：

    ```tsx
    <div
      className={classnames(
        style.wrapper,
        scheme === EColorScheme.DARK && style.darkModeWrapper
      )}
      >
      {/* some code here */}
    </div>
    ```

- 对于深色模式直接引用不同的CSS资源文件
- 用一些第三方的库，比如`postcss-darkmode`等

回到上面话题，通过JS可以监听到系统的颜色主题，那怎么实现用户主动选择颜色主题，不随系统的改变呢？其实也很简单，可以在本地store中设置一个颜色主题的值，用户设置了就优先选用store里的，没有设置就跟随系统，以上面的hooks为例：

```typescript
export const useColorScheme = (): EColorScheme => {
  // 从 store 中取出用户手动设置的主题
  const manualScheme = useSelector(selectManualColorScheme);
  const [scheme, setScheme] = useState(
    darkModeMediaQuery.matches ? EColorScheme.DARK : EColorScheme.LIGHT
  );

  useEffect(() => {
    const listener = (event: MediaQueryListEvent) => {
      setScheme(event.matches ? EColorScheme.DARK : EColorScheme.LIGHT);
    };
    darkModeMediaQuery.addEventListener('change', listener);
    return () => {
      darkModeMediaQuery.removeEventListener('change', listener);
    };
  }, []);

  // 优先取用户手动设置的主题
  return manualScheme || scheme;
};
```

## React Native中的适配

上面说的都是在浏览器里对深色模式的适配，那在React Native里面要怎么适配深色模式呢？

### 1. 大于等于0.62的版本

[Appearance · React Native](https://reactnative.dev/docs/appearance)

在React Native 0.62版本中，引入了`Appearance`模块，通过这个模块：

```typescript
type ColorSchemeName = 'light' | 'dark' | null | undefined;

export namespace Appearance {
  type AppearancePreferences = {
    colorScheme: ColorSchemeName;
  };

  type AppearanceListener = (preferences: AppearancePreferences) => void;

  /**
   * Note: Although color scheme is available immediately, it may change at any
   * time. Any rendering logic or styles that depend on this should try to call
   * this function on every render, rather than caching the value (for example,
   * using inline styles rather than setting a value in a `StyleSheet`).
   *
   * Example: `const colorScheme = Appearance.getColorScheme();`
   */
  export function getColorScheme(): ColorSchemeName;

  /**
   * Add an event handler that is fired when appearance preferences change.
   */
  export function addChangeListener(listener: AppearanceListener): EventSubscription;

  /**
   * Remove an event handler.
   */
  export function removeChangeListener(listener: AppearanceListener): EventSubscription;
}

/**
 * A new useColorScheme hook is provided as the preferred way of accessing
 * the user's preferred color scheme (aka Dark Mode).
 */
export function useColorScheme(): ColorSchemeName;
```

通过`Appearance`模块，可以获得当前的系统颜色主题：

```typescript
const colorScheme = Appearance.getColorScheme();
if (colorScheme === 'dark') {
  // dark mode
} else {
  // light mode
}

Appearance.addChangeListener((prefer: Appearance.AppearancePreferences) => {
  if (prefer.colorScheme === 'dark') {
    // dark mode
  } else {
    // light mode
  }
});
```

同时也提供了一个上面我们自己实现的hooks，`useColorScheme`：

```typescript
const colorScheme = useColorScheme();
```

#### 一些坑

1. `Appearance`这个接口在Chrome调试模式下，会不生效，永远返回`light`

    [Appearance.getColorScheme() always returns ‘light’](https://github.com/facebook/react-native/issues/29144)

1. `Appearance`想要生效，还需要Native做一些配置

    [React Native 0.62.2 Appearance return wrong color scheme](https://stackoverflow.com/questions/61124229/react-native-0-62-2-appearance-return-wrong-color-scheme)

    > Also make sure you do **not** have UIUserInterfaceStyle set in your Info.plist. I had it set to 'light' so Appearance.getColorScheme() was always returning 'light'.

### 2. 小于0.62的版本

对于0.62之前的版本，由于RN没有提供官方接口，需要通过第三方的库`react-native-dark-mode`来实现：
[GitHub - codemotionapps/react-native-dark-mode: Detect dark mode in React Native](https://github.com/codemotionapps/react-native-dark-mode)

它的实现原理感兴趣的可以看下：

> **react-native-dark-mode 实现原理**(这段实现原理其实也是问Chat GPT得到的答案😂)
>
> `react-native-dark-mode`库的实现原理比较简单，它主要是利用了原生平台的接口来检测当前系统是否处于深色模式。在iOS平台上，它使用了`UIUserInterfaceStyle`接口来获取当前系统的界面风格，然后判断是否为暗黑模式。在Android平台上，它使用了`UiModeManager`接口来获取当前系统的 UI 模式，然后判断是否为夜间模式。
>
> 具体来说，`react-native-dark-mode`在React Native项目中提供了一个名为`useDarkMode`的 React Hooks，用于获取当前系统是否处于深色模式。当使用这个Hooks时，它会首先检测当前平台是否支持暗黑模式，如果支持，就直接调用原生平台的接口获取当前系统的界面风格或UI模式，并将结果返回给调用方。如果不支持，就返回一个默认值（比如浅色模式）。
>
> 需要注意的是，由于`react-native-dark-mode`是一个纯JS库，它无法直接调用原生平台的接口。所以它在Native端编写了一个名为`DarkMode`的模块，在JS层通过`NativeModules.DarkMode`来调用。
>
> - 在iOS上，`DarkMode`模块会通过`RCT_EXPORT_MODULE()`宏将自己暴露给RN的JS层。同时，它还会使用`RCT_EXPORT_METHOD()`宏将检测系统界面风格的方法暴露给JS层，使得在JS中可以直接调用该方法。
> - 在Android上，`DarkMode`模块同样会通过`@ReactModule`注解将自己暴露给JS层。然后，它会创建一个名为`DarkModeModule`的Java类，并在该类中实现检测系统UI模式的方法。最后，它会使用`@ReactMethod`注解将该方法暴露给JS层，使得在JS中可以直接调用该方法。

## 参考链接

- [web深色模式适配指南 - 掘金](https://juejin.cn/post/7044043307340529694#heading-8)
- [扫盲， H5适配暗黑主题（DarkMode）全部解法 - 掘金](https://juejin.cn/post/6844904024085364750)
- [紧跟潮流学设计：深色模式设计的8个小技巧-36氪](https://36kr.com/p/1724109946881)
- [即刻7.0：如何设计深色模式？（非官方） | 人人都是产品经理](https://www.woshipm.com/pd/4068702.html)
- [【草稿】深色模式在Web端的适配技巧，附带小程序侧的思考 · Hi头像](https://www.xiaoxili.com/hi-face/docs/other/minapp-to-dark-mode.html)
- [在 React Native 中检测并适配暗黑模式 - 长跑茗](https://www.cpming.top/p/detect-dark-mode-in-rn)
- [Improved dark mode default styling with the color-scheme CSS property and ...](https://web.dev/color-scheme/)
- [Appearance.getColorScheme() always returns ‘light’](https://github.com/facebook/react-native/issues/29144)
- [React Native 0.62.2 Appearance return wrong color scheme](https://stackoverflow.com/questions/61124229/react-native-0-62-2-appearance-return-wrong-color-scheme)
