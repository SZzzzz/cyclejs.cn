# 起步

## 试试 create-cycle-app

创建一个 Cycle.js 项目最快的方法就是使用 [create-cycle-app](https://github.com/cyclejs-community/create-cycle-app)，它提供了 ES6 或 TypeScript、Browserify 或 Webpack 的选项。

> [create-cycle-app >](https://github.com/cyclejs-community/create-cycle-app)

在第一次时需要把 `create-cycle-app` 全局安装到你的系统当中：

```bash
npm install --global create-cycle-app
```

然后，用下面这个命令创建一个名为 *my-awesome-app* （也可以换成你自己想写的名字）的项目，它会带有 Cycle *Run* 和 Cycle *DOM*。

```
create-cycle-app my-awesome-app
```

如果你想用 TypeScript 的话就加一个 `--falvor one-fits-all`。
```
create-cycle-app my-awesome-app --flavor cycle-scripts-one-fits-all
```

## 用 npm 安装

如果你想对你的项目有更多的控制权，推荐使用 [npm](http://npmjs.org/) 来下载安装 Cycle.js。

创建一个新目录并在这个目录中使用这条 npm 命令，它会安装 [xstream](http://staltz.com/xstream)、Cycle *Run* 及 Cycle *DOM*：

```
npm install xstream @cycle/run @cycle/dom
```

*xstream* 和 *Run* 这两个包是运行 Cycle.js 最起码需要安装的 API。*Run* 仅包含了一个函数 `run()`，而 Cycle *DOM* 是一个标准的 DOM Driver，提供一个操作 DOM 的方法；对于数据流（Stream）操作库，你还可以使用其它像 RxJS 或 Most.js 之类的。

> 如果你不知道该怎么去选择，我们推荐使用 xstream。

```
npm install xstream @cycle/run
```

> [RxJS](http://reactivex.io/rxjs)

```
npm install rxjs @cycle/rxjs-run
```

> [Most.js](https://github.com/cujojs/most)

```
npm install most @cycle/most-run
```

注意：像 `@org/package` 这种类型的包，是 [npm scoped packages](https://docs.npmjs.com/getting-started/scoped-packages)，需要你的 npm 版本在 2.11 或以上。你可以通过 `npm --version` 检查是否需要升级 npm 以安装 Cycle.js。

如果你不是在用 DOM 接口构建 Web 应用，你可以不用安装 `@cycle/dom`。

## 编码

我们建议使用一个打包工具，比如 [browserify](http://browserify.org/) 或 [webpack](http://webpack.github.io/)；以及一些转换器用来支持 ES6（或者叫 ES2015），比如 [Babel](http://babeljs.io/) 或 [TypeScript](http://typescriptlang.org/)。本文档大部分的示例代码都是用 ES6 语法的。

### 导入库

当你的构建系统安装好之后，可以像下面这样来开始编写你的 JavaScript 主文件。第二行导入了 `fun(main, drivers)` 函数，这里的 `main` 是我们整个应用的入口，而 `drivers` 则是记录了一些 driver 函数的键值对。

```js
import xs from 'xstream';
import {run} from '@cycle/run';
import {makeDOMDriver} from '@cycle/dom';

// ...
```

### 创建 `main` 和 `drivers`

然后，编写一个 `main` 函数，现在咱们暂时不写内容。`makeDOMDriver(container)` 来自 Cycle *DOM*，它会返回一个用来操作 DOM 的 driver 函数，这个函数我们把它放到 `drivers` 的 `DOM` 字段上。

```js
function main() {
  // ...
}

const drivers = {
  DOM: makeDOMDriver('#app')
};
```

然后，调用 `run()` 去连接 `main` 函数和 `drivers`。

```js
run(main, drivers);
```

### 从 `main` 发消息

我们在 `main()` 函数里写了一些代码：返回一个 `sinks` 对象，`sinks` 对象里面有一个 `xstream` 的数据流挂了在 `DOM` 这个字段上。这里我们能看到 `main()` 正在将一个数据流发送作为消息发送给 DOM driver。Sinks 就是我们要发送的消息。这个数据流发送了一个 Virutal DOM，里面有一个显示着 `${i} seconds elapsed` 的 `<h1>` 元素。这里的 `i` 每秒都会变，会被 `${i}` 替换成 `0`、`1` 和 `2` 之类的。

```js
function main() {
  const sinks = {
    DOM: xs.periodic(1000).map(i =>
      h1('' + i + ' seconds elapsed')
    )
  };
  return sinks;
}
```

对了，记得把 `h1` 导入到 Cycle DOM 中。

> 像这么写:

```js
import {makeDOMDriver, h1} from '@cycle/dom';
```

### 捕获消息到 `main` 中

`main()` 函数现在从拿到 `sources` 作为输入。跟输出的 `sinks` 一样，输入的 `sources` 使用的是同样的数据结构：一个包括了 `DOM` 字段的对象。`sources` 是一些输入的消息，`sources.DOM` 是一个可以可查询（queryable）并得到 Stream 的 API。。使用 `sources.DOM.select(selector).events(eventType)` 可以得到一个数据流，这个数据里面是一些我们用 `eventType` 指定的这类型的事件，来自我们使用 `selector` 指定的元素上。在下面的代码中，我们把来自 `input` 元素 `click` 事件的数据流中的每个数据映射到一个 Virtual DOM 上，以显示一个可以切换状态的复选框（checkbox）。

```js
function main(sources) {
  const sinks = {
    DOM: sources.DOM.select('input').events('click')
      .map(ev => ev.target.checked)
      .startWith(false)
      .map(toggled =>
        div([
          input({attrs: {type: 'checkbox'}}), 'Toggle me',
          p(toggled ? 'ON' : 'off')
        ])
      )
  };
  return sinks;
}
```

别忘了从 Cycle DOM 中导入一些新类型的元素。

> 像这么写：

```js
import {makeDOMDriver, div, input, p} from '@cycle/dom';
```

### 试试 JSX

我们刚刚使用了 `div()`、`input()`、`p()` 函数去创建 Vitual DOM 元素，以得到我们期望的 `<div>`、`<input>` 和 `<p>` DOM 元素，但其实我们还可以用 Babel 来支持 JSX 语法。下面只有在你使用 Babel 来构建时才可以用：


(1) 安装 npm 包 —— [babel-plugin-transform-react-jsx](http://babeljs.io/docs/plugins/transform-react-jsx/) 和 [snabbdom-jsx](https://www.npmjs.com/package/snabbdom-jsx)：

```
npm install --save babel-plugin-transform-react-jsx snabbdom-jsx
```

(2) 在 `.babelrc` 文件中配置支持 JSX：

> .babelrc

```json
{
  "presets": [
    "es2015"
  ],
  "plugins": [
    "syntax-jsx",
    ["transform-react-jsx", {"pragma": "html"}]
  ]
}
```

(3) 引入 Snabbdom JSX：

> main.js

```js
import xs from 'xstream';
import {run} from '@cycle/xstream-run';
import {makeDOMDriver} from '@cycle/dom';
import {html} from 'snabbdom-jsx';
```

(4) 使用 JSX 作为返回值：

这个例子包含了 Cycle.js 中最通用的问题处理方式：明确将电脑的行为表示成一个数据流函数——不断地从 driver 中监听消息和提供给 driver 提供下一个消息（在我们的例子中就是 Virtual DOM 元素）。阅读下一章可以理解这个模式。

```jsx
function main(sources) {
  const sinks = {
    DOM: sources.DOM.select('input').events('click')
      .map(ev => ev.target.checked)
      .startWith(false)
      .map(toggled =>
        <div>
          <input type="checkbox" /> Toggle me
          <p>{toggled ? 'ON' : 'off'}</p>
        </div>
      )
  };
  return sinks;
}
```

## 不使用 npm 来安装

在一些罕见的场合你会需要一个单独的 Cycle.js 脚本文件，你可以在 [unpkg](https://unpkg.com) 上下载它们并马上使用到你的 HTML 文件当中：

- 最新的 Cycle.js [run](https://unpkg.com/@cycle/run/dist/cycle-run.js)
- 最新的 Cycle.js [most.js run](https://unpkg.com/@cycle/most-run/dist/cycle-most-run.js)
- 最新的 Cycle.js [RxJS run](https://unpkg.com/@cycle/rxjs-run/dist/cycle.js)
- 最新的 Cycle.js [DOM](https://unpkg.com/@cycle/dom/dist/cycle-dom.js)
- 最新的 Cycle.js [HTTP](https://unpkg.com/@cycle/http/dist/cycle-http-driver.js)
- 最新的 Cycle.js [Isolate](https://unpkg.com/@cycle/isolate/dist/cycle-isolate.js)
