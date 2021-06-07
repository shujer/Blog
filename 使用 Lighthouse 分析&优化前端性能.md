[lighthouse](https://github.com/GoogleChrome/lighthouse) 是 Google Chrome 推出的一款开源自动化工具，它可以搜集多个现代网页性能指标，分析 Web 应用的性能并生成报告，为开发人员进行性能优化的提供了参考方向。

# 🙈 使用入门

## 在 Chrome DevTools 中使用

比较推荐的方法。我们可以直接打开调试面板中的 “Lighthouse”面板，然后点击生成报告：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1621760580946-c68f65ad-bf29-4283-a603-5586f371b6bd.png#clientId=u1a9b7cf7-2606-4&from=paste&id=ud571ede6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=516&originWidth=1366&originalType=binary&size=71224&status=done&style=none&taskId=u21d55b0f-c034-4b04-9fd9-e932b78aba6#id=bFlFJ&originHeight=516&originWidth=1366&originalType=binary&status=done&style=none)​

## 使用 Chrome 插件

我们可以在[应用商城](https://chrome.google.com/webstore/detail/lighthouse/blipmdconlkpinefehnmjammfjpmpbjk)下载并将插件添加到浏览器，点击插件面板开始分析，使用方法和上面类似。

## 在 Node 命令行工具中使用

在调试面板和插件中只能进行一些基本的配置，如果我们想要更加灵活地配置分析的内容，我们可以在 Node 命令行工具中使用：

- **安装**

```bash
npm install -g lighthouse
# or use yarn:
# yarn global add lighthouse
```

- **分析**：以 `www.baidu.com` 为例，使用以下命令可以生成一份中文报告：

```bash
lighthouse https://www.baidu.com/ --locale=zh-CN --preset=desktop --disable-network-throttling=true --disable-storage-reset=true
```

![report.PNG](https://cdn.nlark.com/yuque/0/2021/png/474741/1622035220149-0901c93e-e688-446e-a8ab-66bfd264d398.png#clientId=u67c54281-db1f-4&from=ui&id=ue2433007&margin=%5Bobject%20Object%5D&name=report.PNG&originHeight=1170&originWidth=1404&originalType=binary&size=143470&status=done&style=none&taskId=u191e8a41-504d-483e-bf81-6db23e90bdd)

- **登录验证**：有的时候，我们需要拿到权限才能访问页面，这个时候，直接执行以上的命令会报错或者分析到的不是目标页。这里，需要我们手动登录或者借助 Puppeteer 模拟登录。参考[文档](https://github.com/GoogleChrome/lighthouse/blob/master/docs/readme.md#testing-on-a-site-with-authentication)，我们可以这样处理：

  - 运行 `chrome-debug` 启动一个 Chrome 实例（全局安装了 lighthouse 之后）
  - 访问我们的目标网址并完成登录
  - 在新的终端，运行上面的命令 `lighthouse [https://example.com](https://www.baidu.com/) --preset=desktop --port=chrome.port`

- 更多的配置见 [cli-options](https://github.com/GoogleChrome/lighthouse#cli-options)

## 使用 Node module

我们还可以在自己的本地项目中，将 Lighthouse 作为一个 [module](https://github.com/GoogleChrome/lighthouse/blob/master/docs/readme.md#using-programmatically) 使用：

- **安装**

```bash
yarn add --dev lighthouse
```

- **自定义启动**

```javascript
const fs = require("fs");
const lighthouse = require("lighthouse");
const chromeLauncher = require("chrome-launcher");

const launchChromeAndRunLighthouse = async (url, opts, config = null) => {
  const options = {
    logLevel: "info",
    output: "html",
    onlyCategories: ["performance"],
    ...opts,
  };
  // 打开 chrome debug
  const chrome = await chromeLauncher.launch({ chromeFlags: opts.chromeFlags });
  // 开始分析
  const runnerResult = await lighthouse(
    url,
    { ...options, port: chrome.port },
    config
  );
  await chrome.kill();
  // 生成报告
  const reportHtml = runnerResult.report;
  fs.writeFileSync("lhreport.html", reportHtml);
};

// 启动
const run = async () => {
  const opts = {
    chromeFlags: ["--show-paint-rects"],
    preset: "desktop",
  };
  await launchChromeAndRunLighthouse("https://example.com", opts);
};
run();
```

## 使用 Lighthouse CI

如果想在每次提交代码/打包构建时生成分析报告，我们可以从考虑接入 [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) 。使用方法很简单，以 `GitHub Action` 为例，我们只需要在项目下添加 ` .github/workflows/lighthouseci.yml`:

```js
name: CI
on: [push]
jobs:
  lhci:
    name: Lighthouse
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: npm install, build
        run: |
          npm install
          npm run build
      - name: run Lighthouse CI
        run: |
          sudo npm install -g @lhci/cli@0.7.x
          lhci autorun
```

如果想自定义一些配置，我们可以添加配置文件 `lighthouserc.js`。之后，我们可以在 Action 面板看到运行的[结果](https://github.com/shujer/lh-ci-demo/actions)：

![](https://cdn.nlark.com/yuque/0/2021/png/474741/1623072490747-0654977f-302e-46e1-91d6-a28ef6794448.png)

# 🙊 性能指标

## 用户关心什么

性能评估的最终目的是提升用户的使用体验。因此，性能指标的制定也要从用户的角度出发。通常来说，从网站加载到可以正常使用，我们需要关注以下几个关键节点的用户感知体验：
![user-centric.PNG](https://cdn.nlark.com/yuque/0/2021/png/474741/1621954869538-ea5e9c3d-f94a-4b2d-893f-925c6c76f65c.png#clientId=u9f7d0d73-8ddb-4&from=ui&id=u5262784c&margin=%5Bobject%20Object%5D&name=user-centric.PNG&originHeight=448&originWidth=1434&originalType=binary&size=132863&status=done&style=none&taskId=u34927b5c-18f8-4ff0-83d1-70d3ddf7d86)

- 根据地址是否能正确访问到网页？服务器请求是否正常响应？
- 页面是否渲染了用户可用的、关心的内容？
- 页面是否可以正常响应用户的交互操作？
- 用户的交互响应是否流畅自然？

从这些角度出发，我们在 [这里](https://googlechrome.github.io/lighthouse/scorecalc/#FCP=0&SI=0&LCP=0&TTI=0&TBT=0&CLS=0&FMP=0&FCI=0&device=desktop&version=6) 看下 Lighthouse 提供的几个性能评估指标：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1621778278882-c65aa1f8-bb3b-44f2-b6b3-2648ed0d7ec2.png#clientId=u1a9b7cf7-2606-4&from=paste&height=398&id=ud278d923&name=image.png&originHeight=796&originWidth=1401&originalType=binary&size=100141&status=done&style=none&taskId=ub3ebe517-fc15-43cd-ab24-847fd3871e8&width=700.5#id=ttu3y&originHeight=796&originWidth=1401&originalType=binary&status=done&style=none)
Lighthouse 最新版的提供了 6 个性能指标：**FCP**、**SI**、**LCP**、**TTI**、**TBT** 和 **CLS**；权重分别是 15%，15%，25%，15%，25% 和 5%。Lighthouse 会根据权重计算得到一个分数值。
​

## 内容呈现相关

### FCP

**FCP**（First Contentful Paint）即**首次内容绘制**。它统计的是**从进入页面到首次有 DOM 内容绘制所用的时间**。这里的 DOM 内容指的是文本、图片、非空的 `canvas` 或者 SVG。我们也可以在 Performance 面板看到这个指标：
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1621956147423-ee4d10be-7394-4682-9d3b-707f5045d426.png#clientId=u9f7d0d73-8ddb-4&from=paste&id=u542c1fdc&name=image.png&originHeight=335&originWidth=1260&originalType=binary&size=48734&status=done&style=none&taskId=u436ad652-583b-476a-b7f7-b6c084b26b9)

**FCP** 和我们常说的白屏问题相关，它记录了页面首次绘制内容的时间。一个常见的影响这个指标的问题是：[FOIT（flash of invisible text，不可见文本闪烁问题）](https://web.dev/font-display/)，即网页使用了体积较大的外部字体库，导致在加载字体资源完成之前字体都不可见。可以通过 [font-display API](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) 来控制字体的展示来解决。

但值得注意的是，页面首次绘制的内容可能不是有意义的。比如页面绘制了一个占位的 loading 图片，这通常不是用户所关心的内容。

### LCP

前面我们已经提到了，由于 FCP 指标统计的内容可能不是用户主要关心的，那么我们需要另一个指标来评估。
**​**

**LCP**（Largest Contentful Paint）即**最大内容绘制**。它统计的是**从页面开始加载到视窗内最大内容绘制的所需时间**，这里的内容指文本、图片、视频、非空的 canvas 或者 SVG 等。

在 **LCP** 之前，lighthouse 还使用过 [**FMP**](https://web.dev/first-meaningful-paint/)（**First Meaningful Paint**，首次有意义内容绘制）指标。FMP 是根据布局对象（layout objects）变化最大的时刻来决定的。但是这个指标计算比较复杂，通常和具体的页面以及浏览器的实现相关，这也会导致计算不够准确。比如，用户在某个时刻绘制了大量的小图标。

**Simpler is better！**用户感知网页的加载速度以及当前的可用性，可以简单地用最大绘制的元素来测量。LCP 指向的最大元素通常会随着页面的加载而变化（只在用户交互操作之前），以下是一个网站的示例：
![bsBm8poY1uQbq7mNvVJm.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1621956709011-2f00d58b-5cf1-480e-bb0f-ee4c7f181e3e.png#clientId=u9f7d0d73-8ddb-4&from=ui&id=ud1857909&margin=%5Bobject%20Object%5D&name=bsBm8poY1uQbq7mNvVJm.png&originHeight=486&originWidth=1252&originalType=binary&size=95993&status=done&style=none&taskId=u322af1be-f45d-4f8e-a126-23fc9e21d8d)

### SI

**SI**（Speed Index）即**速度指数**。Lighthouse 会在页面加载过程中捕获视频，并通过 [speedline](https://github.com/paulirish/speedline) 计算视频中帧与帧之间视觉变化的进度，这个指标反映了**网页内容填充的速度**。页面解析渲染过程中，资源的加载和主线程执行的任务会影响到速度指数的结果。
​

### CLS

前面几个指标关注的都是页面呈现的快慢，但是很多时候我们希望页面的视觉呈现保持相对稳定，比如，不会突然插入一张图片或者元素突然发生位移。
​

这个时候，Lighthouse 使用 **CLS**（Cumulative Layout Shift）即**累计布局位移**进行评估。这个指标是通过比较单个元素在帧与帧之间的位置偏移来计算，计算公式是`cls = impact fraction * distance fraction` 。在以下例子中，文本块在两帧之间的 `impact fraction` 是红色框部分，占视窗 75%；`distance fraction` 是蓝色箭头的距离，占视窗 25%；那么最终的分数是 `0.75 * 0.25 = 0.1875` 。
![](https://cdn.nlark.com/yuque/0/2021/png/474741/1622040274017-920b057a-dbcc-4278-8269-2dbaf733efc2.png#clientId=u67c54281-db1f-4&from=paste&height=484&id=ud1e9316c&originHeight=1200&originWidth=1600&originalType=url&status=done&style=none&taskId=u59a98b42-2004-4cca-91b7-0367b03f07f&width=645)

## 用户交互相关

### TTI

**TTI**（Time To Interactive）即**页面可交互的时间**。这个时间的确定需要同时满足以下几个条件：

- 页面开始绘制内容，即 **FCP** 指标开始之后
- 用户的交互可以及时响应：
  - 页面中大部分可见的元素已经注册了对应的监听事件（通常在 DOMContentLoaded 事件之后）
  - 在 **TTI **之后持续 5 秒的时间内无**长任务**执行（没有超过 50 ms 的执行任务 & 没有超过 2 个 GET 请求）

![](https://cdn.nlark.com/yuque/0/2021/png/474741/1622037705332-a0d1b7f3-34bc-404c-9023-22c892e96430.png#clientId=u67c54281-db1f-4&from=paste&height=403&id=u1a2faba1&margin=%5Bobject%20Object%5D&originHeight=960&originWidth=1614&originalType=url&status=done&style=none&taskId=u28c30005-751d-480e-a948-4f2539ba441&width=678)

### TBT

**TBT**（Total Blocking Time）即阻塞总时间，测量的是 **FCP** 与 **TTI** 之间的时间间隔。这个指标反映了用户的交互是否能及时响应。
如果主线程执行了长任务会导致用户的输入无法及时响应。当主线执行的任务所需的时长超过 50ms，我们就认为这是一个长任务（[long task](https://web.dev/custom-metrics/#long-tasks-api)）。假设在主线程上执行了一系列的任务，每个长任务的阻塞时间等于执行时间减去 50 ms，最后可以统计得到一个总的阻塞时间。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622039286483-08b1f797-78fe-4dcd-b9e4-afda486f1de3.png#clientId=u67c54281-db1f-4&from=paste&id=ubefdba9f&name=image.png&originHeight=197&originWidth=1148&originalType=binary&size=11965&status=done&style=none&taskId=u5376a8e6-64ca-4312-8e53-21ffe84d77d)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622039304415-f076c369-30e4-4faf-9b3d-efd2504cc5ff.png#clientId=u67c54281-db1f-4&from=paste&id=u035f5071&margin=%5Bobject%20Object%5D&name=image.png&originHeight=193&originWidth=1113&originalType=binary&size=12856&status=done&style=none&taskId=ub851ea12-7672-4de6-8092-2c2437672d9)

# 🙉 性能优化

我们可以在报告的** Opportunities** 一节看到影响评估的关键原因，并进行一些优化：
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1621958065917-9fe0c007-bbe0-4dbb-971a-a9baed871f8d.png#clientId=u9f7d0d73-8ddb-4&from=paste&id=CYM38&name=image.png&originHeight=511&originWidth=1065&originalType=binary&size=129178&status=done&style=none&taskId=uda0a914c-5ec7-470f-97a6-1a8dc04eaf8)

## 优化 JavaScript 资源加载

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622380264433-a7006cd5-62ef-4421-a1ba-cea6a3e8b322.png#clientId=uf610cbee-8af3-4&from=paste&id=uc43c2174&margin=%5Bobject%20Object%5D&name=image.png&originHeight=172&originWidth=1058&originalType=binary&size=23345&status=done&style=none&taskId=u252eeb64-e00f-4f8a-b624-d61d3026656)
在 lighthouse 的优化建议中，我们可以看到和 JavaScript 资源加载相关的建议有：

- **Reduce unused JavaScript**
- **Minify JavaScript**
- **Remove duplicate modules in JavaScript bundles**

我们可以看到，资源加载的优化通常有几个思路：

- 合理的加载顺序/策略（延迟加载/预先加载）
- 压缩优化资源的体积
- 代码分割 & 公共提取 & 按需加载

### 分析

如何确定页面中加载了哪些不必要的资源呢？我们可以打开 [Chrome Devtool Coverage](https://developer.chrome.com/docs/devtools/command-menu/) 面板，查看当前使用资源的代码覆盖率，红色表示未使用到的代码。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622211853869-232b0cf9-2214-42e3-a9cb-80bc0c9ad0a6.png#clientId=u3d7dd29b-cff7-4&from=paste&height=395&id=u808ead4c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=790&originWidth=1525&originalType=binary&size=126215&status=done&style=none&taskId=ufa9eceb3-e247-4de2-80b0-43be65ff284&width=762.5)
在 webpack 项目里，可使用 webpack 的插件 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer) 来分析，如下我们启动一个本地端口来查看 webpack 的打包情况。

```javascript
new BundleAnalyzerPlugin({
  analyzerMode: "server",
  analyzerHost: "127.0.0.1",
  analyzerPort: 8888,
  reportFilename: "report.html",
  defaultSizes: "parsed",
  openAnalyzer: true,
  statsFilename: "stats.json",
  statsOptions: {
    exclude: ["vendor", "webpack", "hot"],
  },
  excludeAssets: ["webpack", "hot"],
  logLevel: "info",
});
```

打包后会生成如下的报告，我们可以查看模块打包的情况，还可以切换 `Stat` / `Parsed` / `Gizzped` 来查看开启压缩后代码体积的变化：
![3.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622349356665-98695732-0205-43c8-8d40-319cf625eeb5.png#clientId=uf610cbee-8af3-4&from=ui&id=uc4a4601a&margin=%5Bobject%20Object%5D&name=3.png&originHeight=1019&originWidth=1462&originalType=binary&size=387111&status=done&style=none&taskId=u33fc513e-4b5c-44c2-ba3f-dd22724c05c)

### Code Splitting

减少无用的代码，首先我们需要更加精细合理地切分代码。在 webpack 项目中，通常有三种代码分割的技巧：

- **多入口**：基于 [entry](https://webpack.js.org/configuration/entry-context/#entry) 配置，每个入口打包成单独的 bundle

```javascript
const path = require("path");

module.exports = {
  entry: "./src/index.js",
  mode: "development",
  entry: {
    index: "./src/index.js",
    another: "./src/another-module.js",
  },
  output: {
    filename: "main.js",
    filename: "[name].bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
};
```

- **抽离公共代码**：如果多个页面使用了公共的模块，可以通过 [SpitChunksPlugin](https://webpack.js.org/plugins/split-chunks-plugin/) 将代码分割成多个 chunks：

```javascript
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: "async", // 代码分割时只对异步代码生效；all 全部生效；initial 同步代码生效
      minSize: 20000, // 代码分割的最小体积,
      minChunks: 1, // 做代码分割的最少引用次数
      maxAsyncRequests: 30, // 同时加载模块的最大数量
      cacheGroups: {
        venders: {
          chunks: "all",
          test: /[\\/]node_modules[\\/]/, // 将 node_modules 中的代码进行分割
          priority: -10, // 代码分割的优先级
          reuseExistingChunk: true, // 已经打包过的代码直接复用
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true,
        },
      },
    },
  },
};
W;
```

- **动态加载**：动态加载通常是和上面两种方法结合使用。在打包阶段，通过动态 `import` 引入的模块会被 webpack 单独拆分成一个 `chunk`；在运行的阶段，webpack 会通过 `chunkId` 判断脚本是否已加载并缓存过，没有的话再通过 `script` 标签动态插入。在 React 项目中，可以结合 `React.lazy` 和 `Suspense` 进行路由的切分。

```jsx
import React, { Suspense, lazy } from "react";
import { BrowserRouter as Router, Route, Switch } from "react-router-dom";

const Home = lazy(() => import("./routes/Home"));
const About = lazy(() => import("./routes/About"));

const App = () => (
  <Router>
    <Suspense fallback={<div>Loading...</div>}>
      <Switch>
        <Route exact path="/" component={Home} />
        <Route path="/about" component={About} />
      </Switch>
    </Suspense>
  </Router>
);
```

### DCE

> 摇树优化（Tree-Shaking）是用于消除死代码（Dead-Code Elimination）的一项技术。这主要是依赖于 ES6 模块的特性实现的。ES6 模块的依赖关系是确定的，和运行时状态无关，因此可以对代码进行可靠的静态分析，找到不被执行不被使用的代码然后消除。

对于无用的导入和变量，我们很容易通过 IDE 提示发现。而 webpack 会在 `production` 模式下自动进行一些优化，比如移除无用的导出模块：

```javascript
 mode: 'development',
 optimization: {
   usedExports: true,
 },
```

### 代码压缩

[terser-webpack-plugin](https://www.npmjs.com/package/terser-webpack-plugin)（webpack5 自带）可以用来压缩代码和清理无意义的代码：

```javascript
const TerserPlugin = require("terser-webpack-plugin");
module.exports = {
  optimization: {
    minimize: true,
    minimizer: [new TerserPlugin()],
  },
};
```

之前提到的都是在打包阶段进行优化。除此之外，压缩资源体积更加有效的方法是开启 `gzip`，在 nginx 上我们可以很简单地进行配置：

```nginx
gzip  on;
gzip_min_length  1k;
gzip_buffers     4 16k;
gzip_http_version 1.1;
gzip_comp_level 9;
gzip_types       text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php application/javascript application/json;
gzip_disable "MSIE [1-6]\.";
gzip_vary on;
```

### 考虑脚本加载的顺序

当 HTML 解析时遇到 `script` 脚本会暂停 DOM 树的构建，非异步的脚本需要等待浏览器下载、解析、编译和执行，这会导致渲染的延迟。因此，对于页面中脚本的引用我们需要谨慎地权衡：

- 关键的脚本内联使用，减少不必要的网络等待时间
- 其他的脚本在文档底部引入，或者通过 `async/defer` 异步引用

## 优化 CSS 资源加载

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622380229161-7bd47890-db67-4ac6-98f8-aa96c679dcfb.png#clientId=uf610cbee-8af3-4&from=paste&id=u7023adb8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=123&originWidth=1056&originalType=binary&size=15734&status=done&style=none&taskId=u175828e1-0713-49fa-b126-87262e82dd8)
同样地，lighthouse 给出了一些 CSS 资源优化的建议：

- **Reduce unused CSS**
- **Minify CSS**

### 按需加载

在 webpack 中我们可以使用 [mini-css-extract-plugin](https://www.npmjs.com/package/mini-css-extract-plugin)，将 CSS 文件从 JS 文件中单独抽取出来，支持按需加载：

```javascript
 new MiniCssExtractPlugin({
        filename: "[name].[hash:8].css",
        chunkFilename: "[name].[contenthash:8].css",
      }),
```

### 资源压缩

对于 CSS 的压缩，我们可以使用 [optimize-css-assets-webpack-plugin](https://www.npmjs.com/package/optimize-css-assets-webpack-plugin) 移除无用的代码：

```javascript
new OptimizeCssAssetsPlugin({
  assetNameRegExp: /\.optimize\.css$/g,
  cssProcessor: require("cssnano"),
  cssProcessorPluginOptions: {
    preset: ["default", { discardComments: { removeAll: true } }],
  },
  canPrint: true,
});
```

### 代码优化 & 模块化

除了借助工具来优化 CSS 资源，在开发中我们也要注意一些代码的组织和复用，例如下面这个例子：

```css
h1 {
  background-color: #000000;
}
h2 {
  background-color: #000000;
}
// 减少文件代码
h1,
h2 {
  background-color: #000000;
}
```

随着迭代的开发，项目中很可能会冗余了大量不再需要的 CSS 代码。在前端组件化的时代，将样式和组件绑定，使用更加模块化的方案或许是更加有效的策略。我们可以使用一些 CSS-in-JS 的方案，比如 [styled-components](https://github.com/styled-components/styled-components)，也可以了解下让 Facebook 的主页减少了 80% 的 CSS 体积的 [Atomic CSS（原子 CSS）](https://sebastienlorber.com/atomic-css-in-js)。

## 优化图片的加载

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622370915919-d9d04391-962a-4a28-bc53-e34a61d2cdfe.png#clientId=uf610cbee-8af3-4&from=paste&id=uc59dae95&margin=%5Bobject%20Object%5D&name=image.png&originHeight=126&originWidth=1049&originalType=binary&size=15044&status=done&style=none&taskId=ud7c45024-194a-4534-b3ba-e5a6524f6f3)
关于图片的加载，lighthouse 给出了很多方面的建议：

- **Defer offscreen images**
- **Serve images in next-gen formats**
- **Efficiently encode images**
- **Properly size images**

### 图片懒加载

图片的请求和加载通常需要占用较大的资源，特别是在一些以图片展示为主的电商网站上。对于当前页面不可见的图片，我们可以采用懒加载的方式进行处理。

- 我们可以通过 [IntersectionObserver API](https://developer.mozilla.org/zh-CN/docs/Web/API/IntersectionObserver) 观察图片是否进入可视区域，通常需要 polyfill 处理

```javascript
// html
<img data-src="http://example.com" />;
// js
const lazyLoadIntersection = new IntersectionObserver((entries) => {
  entries.forEach((item, index) => {
    if (item.isIntersecting) {
      item.target.src = item.target.dataset.src;
      lazyLoadIntersection.unobserve(item.target);
    }
  });
});
const imgList = document.querySelectorAll(".img-list img");
Array.from(imgList).forEach((item) => {
  lazyLoadIntersection.observe(item);
});
```

- 我们也可以通过原生 JS 实现：

```javascript
const isInView = (el) => {
  const { top, left, height, width } = el.getBoundingClientRect();
  const clientHeight =
    window.innerHeight || document.documentElement.clientHeight;
  const clientWidth = window.innerWidth;
  document.documentElement.clientWidth;
  if (top < clientHeight && left < clientWidth) {
    return true;
  }
  return false;
};
const lazyload = () => {
  const imgLsit = document.querySelectorAll(".img-list img");
  if (!imgLsit.length) {
    document.removeEventListener("scroll", throttleScroll);
    return;
  }
  imgLsit.forEach((img) => {
    if (!img.src && isInView(img)) {
      img.src = img.dataset.src;
    }
  });
};
lazyload();
const throttleScroll = throttle(lazyload, 300);
document.addEventListener("scroll", throttleScroll, { passive: true });
```

### 响应式图片

除了懒加载，在一些响应式的网站，我们往往希望图片可以根据设备型号的不同，加载不同分辨率和大小的图片，这可以使用 [Responsive images](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images) 方案来解决。主要是基于 `srcset`属性：

```javascript
<img srcset="elva-fairy-480w.jpg 480w,
             elva-fairy-800w.jpg 800w"
     sizes="(max-width: 600px) 480px,
            800px"
     src="elva-fairy-800w.jpg"
     alt="Elva dressed as a fairy">
```

### WebP 图片格式优化

WebP 是由谷歌（[Google](https://developers.google.com/speed/webp/)）推出的一种旨在加快图片加载速度的图片格式，在相同质量的情况下，WebP 的体积要比 JPEG 格式小 25% ~ 34%。目前，一些图片依托站点也提供了 webp 的支持，我们在使用之前需要检测当前浏览器的兼容情况。

```javascript
<picture>
  <source type="image/svg+xml" srcset="pyramid.svg">
  <source type="image/webp" srcset="pyramid.webp">
  <img src="pyramid.png" alt="regular pyramid built from four equilateral triangles">
</picture>
```

## 优化网络请求

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622380915922-318dafe2-edfd-4207-a565-0c059f1d5bb6.png#clientId=uf610cbee-8af3-4&from=paste&id=u12c15bd8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=98&originWidth=1055&originalType=binary&size=9495&status=done&style=none&taskId=u1ce11278-1fed-4c47-815d-e8bcf801d22)
针对网络连接的建立和请求的缓存，lighthouse 也给出了一些参考的建议：

- **Use HTTP/2**
- **Serve static assets with an efficient cache policy**
- **Preload key requests**
- **Preconnect to required origins**
- **Avoid enormous network payloads**
- **Fonts with font-display: optional are preloaded**
- **Preload Largest Contentful Paint image**

### CDN 加速

我们可以将一些资源，比如图片、公共类库放到 CDN 上，基于内容分发和缓存提高响应速度，阿里云的 OSS 还提供了图片的多格式转换。

> **内容分发网络**（Content Delivery Network，简称 CDN）是由分布在不同区域的边缘节点服务器群组成的。CDN 可以将源站内容分发到距离用户最近的节点，从而提高响应速度和成功率。其底层是依赖 DNS（域名解析服务）返回给用户最近的 IP 地址实现的。

### 合理的 HTTP 缓存策略

对于放置在 CDN 的资源，我们需要设置合理的 HTTP 缓存策略来资源的命中和加载。HTTP 有两种缓存机制，强缓存（基于 Expires 和 Cache-Control）以及协商缓存（基于 Last-Modified/if-Modified-Since 或 Etag / If-None-Match）。

- 对于一些不需要经常变化的资源，或者 webpack 中每次打包会根据内容生成文件路径（`contenthash`）的资源，我们可以设置一个比较大的缓存时长或者过期时间
- 而对于首页 `index.html` 等页面我们应该禁止缓存，或者设置一个比较短的缓存时间并检查资源的新鲜度

### 使用 HTTP/2

我们知道在 HTTP/1.1 中，浏览器对于同一个域名的 tcp 连接数有限制，通常只能同时发起 6 ~ 8 个连接。同时发起多个请求，在 Network 面板我们可以看到明显的阶梯式变化以及很多的 Queueing 排队等待的时间：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622385872071-1ea952eb-07d1-49f0-827d-c9b31d0599c3.png#clientId=uf610cbee-8af3-4&from=paste&id=u30b4a617&margin=%5Bobject%20Object%5D&name=image.png&originHeight=642&originWidth=1011&originalType=binary&size=219499&status=done&style=none&taskId=u10881c67-44db-4aeb-a31d-49536fe73bc)
HTTP2 增加了多路复用、流优先级、头部压缩等功能，可以很好地解决 tcp 连接限制和队头阻塞问题。我们可以在 nginx 在配置，需要结合 https 使用：

```nginx
listen 443 ssl http2;
```

### 使用 pre-\* 预处理

- **preload**：通过 `rel="preload` 对优先级较高的资源进行内容预加载，比如一些重要的样式、字体文件和图片，我们不希望页面在显示的时候出现闪动，可以预加载；也可以使用 [preload-webpack-plugin](https://github.com/GoogleChromeLabs/preload-webpack-plugin)；

```html
<link rel="preload" href="main.js" as="script" />
<link
  rel="preload"
  href="fonts/cicle_fina-webfont.ttf"
  as="font"
  type="font/ttf"
  crossorigin="anonymous"
/>
```

- **dns-prefetch**：当页面通过网址请求服务器的时候，需要先通过 DNS 解析拿到 IP 地址才能发起请求。如果网站存在大量跨域的资源，DNS 的解析过程很可能会降低页面的性能。对于关键的跨域资源，我们最好进行 dns 预获取，还可以结合 `preconnect` 进行预连接：

```html
<link rel="preconnect" href="https://fonts.gstatic.com/" crossorigin />
<link rel="dns-prefetch" href="https://fonts.gstatic.com/" />
```

- **prefetch**: `preload` 通常用于预加载当前页面的一些关键资源，如果我们想预获取将要用到的资源或将要导航到的页面，可以使用 `prefetch`；

## 优化页面渲染性能

![image.png](https://cdn.nlark.com/yuque/0/2021/png/474741/1622468324353-ba1f7e8b-d48f-4b36-a9c0-7a7d59396a64.png#clientId=ue7395e69-f0de-4&from=paste&height=265&id=u86f62937&margin=%5Bobject%20Object%5D&name=image.png&originHeight=289&originWidth=624&originalType=url&size=75910&status=done&style=none&taskId=u9c4e7031-22e0-4351-ad2b-c296af1dfcf&width=572)

我们知道，浏览器的页面渲染通常需要经过构建 **DOM 树**，构建** CSSOM 树**，构建**渲染树**、样式计算和**布局**、**绘制**、**合成**帧并绘制到屏幕几个过程。在这个过程中，主线程复杂的 JS 任务会阻塞渲染。
​

针对解析和渲染的过程，lighthouse 也提出了一些优化建议：

- **Avoid an excessive DOM size**
- **Avoid large layout shifts**
- **Avoid non-composited animations**
- **Image elements do not have explicit width and height**

### 高性能的动画

![frame-no-layout-paint.jpg](https://cdn.nlark.com/yuque/0/2021/jpeg/474741/1622468150274-dc4e0e54-d8f9-49e8-aac0-6e0111974c0b.jpeg#clientId=ue7395e69-f0de-4&from=ui&id=ud927f617&name=frame-no-layout-paint.jpg&originHeight=167&originWidth=1093&originalType=binary&size=9940&status=done&style=none&taskId=u9442078d-0f7e-4d5f-8720-50c7931fc57)
当我们进行一些样式变换，比如改变元素的宽高时，浏览器需要重新计算元素的几何元素和样式，这个过程可能需要耗费一些时间。合成线程可以开启硬件加速且无需等待主线程计算，因此，动画应该尽可能放到合成线程处理。通常有几种方法：

- 使用 `transform: scale()` 代替 `width` 和 `height`
- 使用 `transform: translate()` 代替 `top`、`right`、`bottom`、`left` 属性

同时，要保证这些元素存在合成图层中，通常可以通过提升元素（在父元素中声明）来解决：

- ​`will-change: transform`
- `transform: translate3d(0, 0, 0)`

### 避免执行长任务

- 通过时间分片（Time Slicing），将复杂的任务拆分成多个异步执行的任务
- 将复杂的任务放到 worker 线程中，计算有了结果再通知主线程

# 参考资料

[lighthouse](https://github.com/GoogleChrome/lighthouse)
[Lighthouse performance scoring](https://web.dev/performance-scoring/)​
[webpack](https://webpack.js.org/)
[HTTP/2: the difference between HTTP/1.1, benefits and how to use it](https://factoryhr.medium.com/http-2-the-difference-between-http-1-1-benefits-and-how-to-use-it-38094fa0e95b)
[How Browsers Work: Behind the scenes of modern web browsers](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)​
