# 组件

## 自动可重用

UI 通常由许多可重用的元素组成：按钮，图表，滑块，划过时拥有效果的头像，智能表单域等等。包含 Cycle.js 在内的许多前端框架中，这些元素被称为组件。但在 Cycle.js 中，组件还具有一个特性。

**任意 Cycle.js 应用都能够被作为一个组件重用到更大的一个 Cycle.js 应用中。**

这个特性是什么意思呢？它意味着你可以构建一个应用程序，它简单到只是制作一个滑块组件，这个组件在前文中我们已经用 Cycle.js `main()` 实现了。之后，由于 `main()` 仅只是一个函数，它从外部世界获得输入，并通过 return 来产出输出，在一个更大的 Cycle.js 应用中，我们只需要调用这个 `main()`，就能获得一个智能滑块。

每一个“小的 Cycle.js `main()` 函数”都被称为 **数据流组件（dataflow component）**。一个数据流组件所接收的 sources 都是其父组件提供的流（streams），该组件返回的 sinks 则会回馈给父组件。在前文中，我们已经构建了数据流组件，传入 `run(main, drivers)` 的 `main()` 也是一个数据流组件，它的父组件就是各个 drivers，因为正是这些 drivers 给予了该组件需要的 sources，并且拿到了组件输出的 sinks。

![dataflow component](img/dataflow-component.svg)

为了更好的理解 Cycle.js 中的组件，我们干脆亲自做一个数据流组件，这个组件是一个简单的带标签的滑块。该滑块组件接收用户事件作为输入，并输出一个滑块的虚拟 DOM 流，以及一个反映了滑块值的流。该组件也能从父组件接收一些参数来决定自己的行为和展示，在其他框架中，这些参数会被称为 **props** （属性）。

## 栗子：labeled slider 组件

顾名思义，一个 labeled slider 组件由两个部分组成：一个标签（label）和一个滑块（slider），当滑块滑动时，标签能动态显示当前的滑块值。

<a class="jsbin-embed" href="//jsbin.com/napoke/embed?output">JS Bin on jsbin.com</a>

每个 labeled slider 都有下面这些属性：

- 标签文本（`'Weight'`，`'Height'` 等等）
- 单位文本（`'kg'`，`'cm'` 等等）
- 滑块最小值
- 滑块最大值
- 初始值

这些属性可以被编码为一个对象，并使用流进行包裹 `props$`，`main()` 可以通过 `sources.props` 获得属性：

```javascript
function main(sources) {
  const props$ = sources.props;
  // ...
  return sinks;
}
```

之后调用 `run` 来使用这个 main 函数，并传入初始属性：

```javascript
run(main, {
  props: () => xs.of({
    label: 'Weight', unit: 'kg', min: 40, value: 70, max: 140
  }),
  DOM: makeDOMDriver('#app')
});
```

需要牢记的是，即便我们只是在构建一个滑块组件，我们仍需要把构建组件当做是在构建一个 main 程序。接下来，由于 `props` 是 labeled slider 从其父组件所接收的输入，所以 `main()`  的唯一父组件在这个例子中就是 `run`。 正因如此，我们需要将 `props` 配置为一个伪 driver。

另一个 labeled slider 程序需要的输入是 DOM source：

```diff
 function main(sources) {
+  const domSource = sources.DOM;
   const props$ = sources.props;
   // ...
   return sinks;
 }
```

程序剩余部分就很简单了，和之前两章 labeled slider 代码如出一辙，只不过我们需要处理一下传入的初始值 props：

```javascript
function main(sources) {
  const domSource = sources.DOM;
  const props$ = sources.props;

  const newValue$ = domSource
    .select('.slider')
    .events('input')
    .map(ev => ev.target.value);

  const state$ = props$
    .map(props => newValue$
      .map(val => ({
        label: props.label,
        unit: props.unit,
        min: props.min,
        value: val,
        max: props.max
      }))
      .startWith(props)
    )
    .flatten()
    .remember();

  const vdom$ = state$
    .map(state =>
      div('.labeled-slider', [
        span('.label',
          state.label + ' ' + state.value + state.unit
        ),
        input('.slider', {
          attrs: {type: 'range', min: state.min, max: state.max, value: state.value}
        })
      ])
    );

  const sinks = {
    DOM: vdom$,
    value: state$.map(state => state.value),
  };
  return sinks;
}
```

或许你已经留意到了，除了输出一个虚拟 DOM 流，我们也返回了一个 `value$` 流作为一个 sink：

```diff
   // ...
   const sinks = {
     DOM: vdom$,
+    value: value$,
   };
   return sinks;
 }
```

如果 labeled slider 的父组件需要使用到滑块的数值，比如使用身高或者体重数值来计算 BMI（Body Mass Index：体质指数），那么 `value$` 就显得尤为必要了。在上面我们写的程序中，`main()` 的父组件是一些不需要用到 `value$` 的 drivers，因此，我们也不需要一个叫做 `value` 的 driver（即不需要一个处理 `value$` 的 driver）。但是，就像接下来会看到的，如果滑块组件的父组件是另一个数据流组件，`value$` 就很重要。

> 如何命名 sources/sinks ？
>
> 你应该已经发现了我们使用 `value` 而不是 `value$` 来命名 sink。这样的命名方式乍看起来似乎与我们的对流的命名惯例（即每个流都需要跟一个后缀 `$`）相悖。但事实并非如此。
>
> sources 和 sinks 是一个反例，因为二者扮演了 sockets 一般的角色，这些 sockets 连接了组件和外部世界。sources 和 sinks 的名字仅只是作为设置或者获取流的 “键（keys）”。在 `run(main, drivers)` 语句中，这些 keys 需要匹配 `drivers` 对象中的相同的 keys。可以看到，为什么在这些 drivers 对象中，
由某个 key 索引的 driver 都不是一个流。它们是函数，因为 drivers 都是函数。
>
> 这也就是为什么我们不应该使用 `DOM$` 来命名 DOM sink，在 `drivers` 对象中，`DOM` 对应的 driver 是 DOM Driver，该 driver 是一个函数而不是一个流。并且，在 `main` 函数中，`sources.DOM` 是一个 DOM Source 对象（而不是流），它具有像 `select()` 和 `events()` 这样的方法。
>
> 命名 sink 的时候，使用简单的 **key** 来命名就行了，不需要遵从流命名的管理来添加后缀 `$`。之后，我们可以通过类似 `const props$ = sources.props` 的方式来从 sources 中获得我们需要的流。

## 使用组件

现在，labeled slider 这个数据流组件已经整装待发了，我们可以在一个更大型的应用中使用它了。首先，我们先把组件重命名为 `LabeledSlider`，来和组件所处应用中的 `main()` 区分。

```diff
-function main(sources) {
+function LabeledSlider(sources) {
   const domSource = sources.DOM;
   const props$ = sources.props;

   // ...

   return sinks;
 }

+function main(sources) {
+  // Call LabeledSlider() here...
+}
```

由于 `LabeledSlider` 仅仅是一个函数，我们可以传递一些 sources 来调用该函数，获得它输出的 sinks。

```javascript
function main(sources) {
  const props$ = xs.of({
    label: 'Radius', unit: '', min: 10, value: 30, max: 100
  });
  const childSources = {DOM: sources.DOM, props: props$};
  const labeledSlider = LabeledSlider(childSources);
  const childVDom$ = labeledSlider.DOM;
  const childValue$ = labeledSlider.value;
  // ...
}
```

> 为什么使用大写命名法来命名 components ？
>
> 你可能已经注意到了我们将上面的数据流组件命名为了 `LabeledSlider`。在 JavaScript 中，大写命名法通常用于类（class）和构造函数（constructor）的命名。由于 Cycle.js 大量使用了函数式编程技术，面向对象程序的命名惯例就无需在乎了，在 Cycle.js 中几乎甚至完全没有类。
>
> 据此，大写命名法在使用了函数式编程的 JavaScript 中也变得可用。我们不需要认为大写的名词代表了类或者构造函数，在 Cycle.js 中，使用大写命名法命名数据流组件将会是约定俗成的惯例。我们会使用 `FooButton` 来命名数据流组件，其对应的驼峰命名法 `fooButton` 则用于命名该组件输出的 sink 对象。

现在，我们分别使用 `childDom$` 和 `childValue$` 来命名 labeled slider 输出的 sink，方便了我们在父组件 `main()` 中把它们当做流来使用。`childValue$` 能够获得 slider 当前的值，根据这个值，我们将渲染对应半径大小的圆形。而 `childVDom$` 将拿到 slider 对应的虚拟 DOM，以嵌入到父组件的虚拟 DOM 中。

```javascript
function main(sources) {
  // ...

  const childVDom$ = labeledSlider.DOM;
  const childValue$ = labeledSlider.value;

  const vdom$ = xs.combine(childValue$, childVDom$)
    .map(([value, childVDom]) =>
      div([
        childVDom,
        div({style: {
          backgroundColor: '#58D3D8',
          width: String(2 * value) + 'px',
          height: String(2 * value) + 'px',
          borderRadius: String(value) + 'px'
        }})
      ])
    );

  return {
    DOM: vdom$
  };
}
```

最终，我们得到一个 Cycle.js 的应用，在这个应用中，labeled slider 控制了将要渲染的圆形的尺寸。

<a class="jsbin-embed" href="//jsbin.com/yojoho/embed?output">JS Bin on jsbin.com</a>

## 隔离组件实例

现在，我们看看如何使用 labeled slider 来构建 BMI 小程序。

最原始的方式是调用两次 `LabeledSlider()`，一次传入体重需要的 props 来获得体重滑块，一次则是传入身高需要的 props 来获得身高滑块：

<a class="jsbin-embed" href="//jsbin.com/lagegax/embed?output">JS Bin on jsbin.com</a>

```javascript
function main(sources) {
  const weightProps$ = xs.of({
    label: 'Weight', unit: 'kg', min: 40, value: 70, max: 150
  });

  const heightProps$ = xs.of({
    label: 'Height', unit: 'cm', min: 140, value: 170, max: 210
  });

  const weightSources = {DOM: sources.DOM, props: weightProps$};
  const heightSources = {DOM: sources.DOM, props: heightProps$};

  const weightSlider = LabeledSlider(weightSources);
  const heightSlider = LabeledSlider(heightSources);

  const weightVDom$ = weightSlider.DOM;
  const weightValue$ = weightSlider.value;

  const heightVDom$ = heightSlider.DOM;
  const heightValue$ = heightSlider.value;

  const bmi$ = xs.combine(weightValue$, heightValue$)
    .map(([weight, height]) => {
      const heightMeters = height * 0.01;
      const bmi = Math.round(weight / (heightMeters * heightMeters));
      return bmi;
    })
    .remember();

  const vdom$ = xs.combine(bmi$, weightVDom$, heightVDom$)
    .map(([bmi, weightVDom, heightVDom]) =>
      div([
        weightVDom,
        heightVDom,
        h2('BMI is ' + bmi)
      ])
    );

  return {
    DOM: vdom$
  };
}
```

不幸的是，程序出现了 bug。我们滑动任意一个滑块，另一个滑块也在跟着动。我们只有再次进入到 `LabeledSlider` 的实现中去一窥究竟：

```javascript
function LabeledSlider(sources) {
  // ...

  const newValue$ = domSource
    .select('.slider')
    .events('input')
    .map(ev => ev.target.value);

  // ...
}
```

假定我们仅仅调用该函数来获得一个体重滑块。语句 `sources.DOM.select('.slider')` **会在整个 DOM 树中选择所有**  `.slider` 元素。又由于我们的体重和身高组件的 css class 都是 `.slider`，因此，当我们滑动任意滑块时，两个滑块都会动。

各个组件之间应当互不干扰。为了保证 **一个组件仅只是一个 Cycle.js 应用**，我们还需要实现下面两个性质：

- 一个组件的 **sources**（输入）不会受其他组件 sources 的影响，
- 一个组件的 **sinks**（输出）不会受到其他组件 sinks 的影响。

为此，当 sources 传入组件时，当 sinks 从组件返回时，这两个对象我们都会稍加更改。为了让各个组件的 sources 和 sinks 相互隔离，我们需要为每个组件创建各自的 scope。

对于 DOM source 和 DOM sink，我们可以使用一个唯一标识符来作为每个虚拟 DOM 元素的命名空间。首先，我们会为 DOM sink 打个补丁，该流发出的 VNode 会被添加额外的 css class：

```diff
 function main(sources) {
   // ...

   const weightSlider = LabeledSlider(weightSources);
   const heightSlider = LabeledSlider(heightSources);

   const weightVDom$ = weightSlider.DOM
+    .map(vnode => {
+      vnode.sel += '.weight';
+      return vnode;
+    });
   const weightValue$ = weightSlider.value;

   const heightVDom$ = heightSlider.DOM
+    .map(vnode => {
+      vnode.sel += '.height';
+      return vnode;
+    });
   const heightValue$ = heightSlider.value;

   // ...
 }
```

最终渲染的 HTML 如下：

```html
<div class="labeled-slider weight">
  <span class="label">Weight 70kg</span>
  <input class="slider" type="range" min="40" max="150">
</div>
```

滑动体重滑块时，即执行 `sources.DOM.select('.slider').events('input')` 时，**只**应该在 `<div class="labeled-slider weight">` 及其子孙元素上检测用户用户事件。

换言之，在任何一个 labeled slider component 中，**`sources.DOM.select()` 选中的 DOM，应当对应于该组件输出的 DOM sink**。

在 `sources.DOM.select()` 中，使用与对 sink 打补丁时相同的 css class 就能能满足这样的需求。

```diff
 function main(sources) {
   // ...
   const weightSources = {
-    DOM: sources.DOM,
+    DOM: sources.DOM.select('.weight'),
     props: weightProps$
   };
   const heightSources = {
-    DOM: sources.DOM,
+    DOM: sources.DOM.select('.height'),
     props: heightProps$
   };

   const weightSlider = LabeledSlider(weightSources);
   const heightSlider = LabeledSlider(heightSources);
   // ...
 }
```

> ### `select()` 做了些什么？
>
> 在前文中，我们多次使用了 `.select(selector).events(eventType)` 来获得 `selector` 对应元素发出的 `eventType` 类型的 DOM 事件流。
>
> 在上面的代码中，`sources.DOM` 被称为 “DOM source”，该对象绑定了一些函数来帮助我们查询 DOM 以及获得事件流。我们也会直接调用不带 `.events(eventType)` 的 `sources.DOM.select(selector)`，这样，语句会返回一个 **新的** DOM source，我们又可以在这个新的 DOM source 上调用 `select()` 或者 `events` 方法。
>
> `select('.foo').select('.bar').events('click')` 返回了一个发生在 `'.foo .bar'` 元素上的点击事件流。`select('.foo')` “收窄”了 DOM source 的 scope，让我们在更深一层 DOM source 上去寻找点击流。

我们完成的用来隔离不同组件 sources 和 sinks 的代码看起来太过样板（boilerplate）了。理想状态下，我们想要避免手动设置 className 来限制组件的 scope：

```javascript
function main(sources) {
  // ...
  const weightSources = {
    DOM: sources.DOM.select('.weight'), props: weightProps$
  };
  const heightSources = {
    DOM: sources.DOM.select('.height'), props: heightProps$
  };
  // ...
  const weightVDom$ = weightSlider.DOM
    .map(vnode => {
      vnode.sel += '.weight';
      return vnode;
    });
  // ...
  const heightVDom$ = heightSlider.DOM
    .map(vnode => {
      vnode.sel += '.height';
      return vnode;
    });
  // ...
}
```

为了减少代码重复，例如使用 `.map(vnode => ...)` 来为 VNodes 打补丁，我们可以抽象两个函数来分别隔离 DOM sink 和 DOM source：`isolateDOMSink()` ， `isolateDOMSource()`。

```diff
 function main(sources) {
   // ...
   const weightSources = {
-    DOM: sources.DOM.select('.weight'), props: weightProps$
+    DOM: isolateDOMSource(sources.DOM, 'weight'), props: weightProps$
   };
   const heightSources = {
-    DOM: sources.DOM.select('.height'), props: heightProps$
+    DOM: isolateDOMSource(sources.DOM, 'height'), props: heightProps$
   };
   // ...
-  const weightVDom$ = weightSlider.DOM
-    .map(vnode => {
-      vnode.sel += '.weight';
-      return vnode;
-    });
+  const weightVDom$ = isolateDOMSink(weightSlider.DOM, 'weight');
   // ...
-  const heightVDom$ = heightSlider.DOM
-    .map(vnode => {
-      vnode.sel += '.height';
-      return vnode;
-    });
+  const heightVDom$ = isolateDOMSink(heightSlider.DOM, 'height');
   // ...
 }
```

这两个非常有用的帮助函数已经被打包到了 Cycle DOM 中。二者可以被作为静态方法直接使用：`sources.DOM.isolateSource` ， `sources.DOM.isolateSink`。使用了这两个方法以后的 `main()` 函数如下：

```javascript
function main(sources) {
  const weightProps$ = xs.of({
    label: 'Weight', unit: 'kg', min: 40, value: 70, max: 150
  });
  const heightProps$ = xs.of({
    label: 'Height', unit: 'cm', min: 140, value: 170, max: 210
  });

  const {isolateSource, isolateSink} = sources.DOM;

  const weightSources = {
    DOM: isolateSource(sources.DOM, 'weight'), props: weightProps$
  };
  const heightSources = {
    DOM: isolateSource(sources.DOM, 'height'), props: heightProps$
  };

  const weightSlider = LabeledSlider(weightSources);
  const heightSlider = LabeledSlider(heightSources);

  const weightVDom$ = isolateSink(weightSlider.DOM, 'weight');
  const weightValue$ = weightSlider.value;

  const heightVDom$ = isolateSink(heightSlider.DOM, 'height');
  const heightValue$ = heightSlider.value;

  // ...
}
```

上述代码中，我们需要手动每个组件的 sources 和 sinks 来保证它们之间运行在隔离的上下文中。如果我们能直接在底层“隔离”组件的 source 和 sink，就更好了。

这正是函数 [`isolate()`](https://github.com/cyclejs/cyclejs/tree/master/isolate) （`npm install @cycle/isolate`）的由来。该函数内部调用了 `isolateSource` 和 `isolateSink` 。`isolate(Component, scope)` 接受一个数据流组件 `Component` 作为输入，输出一个 sources 和 scope 都被隔离到 `scope` 中的数据流组件。下面的代码块展示了 `isolate()` 的最简实现：

```javascript
function isolate(Component, scope) {
  return function IsolatedComponent(sources) {
    const {isolateSource, isolateSink} = sources.DOM;
    const isolatedDOMSource = isolateSource(sources.DOM, scope);
    const sinks = Component({DOM: isolatedDOMSource});
    const isolatedDOMSink = isolateSink(sinks.DOM, scope);
    return {
      DOM: isolatedDOMSink
    };
  };
}
```

现在，我们可以进一步简化我们的 `main()` 函数了：

```diff
 function main(sources) {
   const weightProps$ = xs.of({
     label: 'Weight', unit: 'kg', min: 40, value: 70, max: 150
   });
   const heightProps$ = xs.of({
     label: 'Height', unit: 'cm', min: 140, value: 170, max: 210
   });

-  const {isolateSource, isolateSink} = sources.DOM;
   const weightSources = {
-    DOM: isolateSource(sources.DOM, 'weight'), props: weightProps$
+    DOM: sources.DOM, props: weightProps$
   };
   const heightSources = {
-    DOM: isolateSource(sources.DOM, 'height'), props: heightProps$
+    DOM: sources.DOM, props: heightProps$
   };

+  const WeightSlider = isolate(LabeledSlider, 'weight');
+  const HeightSlider = isolate(LabeledSlider, 'height');

-  const weightSlider = LabeledSlider(weightSources);
+  const weightSlider = WeightSlider(weightSources);
-  const heightSlider = LabeledSlider(heightSources);
+  const heightSlider = HeightSlider(heightSources);

-  const weightVDom$ = isolateSink(weightSlider.DOM, 'weight');
+  const weightVDom$ = weightSlider.DOM;
   const weightValue$ = weightSlider.value;

-  const heightVDom$ = isolateSink(heightSlider.DOM, 'height');
+  const heightVDom$ = heightSlider.DOM;
   const heightValue$ = heightSlider.value;

   // ...
 }
```

注意到创建 `WeightSlider` 组件的代码：

```javascript
const WeightSlider = isolate(LabeledSlider, 'weight');
```

`isolate()` 接受一个没有被隔离的组件 `LabeledSlider` 并且将其限制到了 `'weight'` scope 中，最终输出了一个隔离好的组件 `WeightSlider`。这个 `'weight'` scope 只在这行代码使用了，其他地方并没有使用。因此，我们还可以通过不显式声明 scope 参数来进一步简化代码：

```javascript
const WeightSlider = isolate(LabeledSlider);
```

如果不显式传递 scope 参数，那么会自动生成一个全局唯一的 scope 字符串，我们不需要关心这个字符串到底是什么，只要知道它确实帮助我们隔离了组件实例就好。

> ### `isolate()` 是否是纯函数？
>
> 如果我们对于身高和体重的滑块组件都不显式声明 scope 参数，那么代码就会是：
>
> `const WeightSlider = isolate(LabeledSlider);`<br />
> `const HeightSlider = isolate(LabeledSlider);`
>
> 由于等号右边的代码都是一样的，是否就意味着 `WeightSlider` 和 `HeightSlider` 是相同的组件？**当然不是这样。**
>
> 没有声明 scope 参数的 `isolate()` **不是** 引用透明（referential transparency）的。换言之，隐式传递 scope 的 `isolate()` 是不纯的。`WeightSlider` 和 `HeightSlider` 不是相同的组件，二者有各自唯一的 scope 参数。  
>
> 另一方面，如果显式传递了 scope 参数，`isolate()` 则是引用透明的，下面的代码中，`Foo` 和 `Fuu` 就是相同的：
>
> `const Foo = isolate(LabeledSlider, 'myScope');`<br />
> `const Fuu = isolate(LabeledSlider, 'myScope');`
>
> 由于 Cycle.js 谨遵函数式编程技术，因此，其大部分 API 都是引用透明的。但为了方便，`isolate()` 则成为了一个例外。如果你追求任何时候任何地点都引用透明，那么就显示传递 scope 参数吧。如果你为了方便，并且知道 `isolate()` 是怎么工作的，就不用传递 scope 参数了。

对比下我们一开始的 BMI 计算器代码，唯一的区别仅仅是现在通过 `isolate()` 进行了组件隔离：

```diff
 function main(sources) {
   const weightProps$ = xs.of({
     label: 'Weight', unit: 'kg', min: 40, value: 70, max: 150
   });
   const heightProps$ = xs.of({
     label: 'Height', unit: 'cm', min: 140, value: 170, max: 210
   });

   const weightSources = {DOM: sources.DOM, props: weightProps$};
   const heightSources = {DOM: sources.DOM, props: heightProps$};

-  const weightSlider =         LabeledSlider(weightSources);
+  const weightSlider = isolate(LabeledSlider)(weightSources);
-  const heightSlider =         LabeledSlider(heightSources);
+  const heightSlider = isolate(LabeledSlider)(heightSources);

   const weightVDom$ = weightSlider.DOM;
   const weightValue$ = weightSlider.value;

   const heightVDom$ = heightSlider.DOM;
   const heightValue$ = heightSlider.value;

   const bmi$ = xs.combine(weightValue$, heightValue$)
     .map(([weight, height]) => {
       const heightMeters = height * 0.01;
       const bmi = Math.round(weight / (heightMeters * heightMeters));
       return bmi;
     })
     .remember();

   const vdom$ = xs.combine(bmi$, weightVDom$, heightVDom$)
    .map(([bmi, weightVDom, heightVDom]) =>
      div([
        weightVDom,
        heightVDom,
        h2('BMI is ' + bmi)
      ])
    );

   return {
     DOM: vdom$
   };
 }
```

需要牢记的是：**当创建相同类型组件的多个实例时，要对每个实例进行 `isolate`。**

<a class="jsbin-embed" href="//jsbin.com/seqehat/embed?output">JS Bin on jsbin.com</a>

## Recap

为了实现组件的可重用性，**任何 Cycle.js 组件都只是一个函数，这个函数可以被用在任何的更大的 Cycle.js 应用中**。sources 和 sinks 是应用和 driver 间的接口，同时也是子组件及其父组件间的接口。

![nested components](img/nested-components.svg)

从组件视角来看，它不用关心它的父组件、或者父亲容器是什么。如果该组件被用作了 `main()`，那么其父组件就是 drivers，当然，父组件也可以是其他的数据流组件。据此，一个组件应当假定它的 sources 只含有与它自己相关的数据。亦即，一个组件的 sources 和 sinks 都需要被 **隔离**。

使用 `isolateSource` 和 `isolateSink` 来隔离兄弟组件或者不相关组件的执行上下文。而使用 `isolate` 来创建一个隔离组件，它的内部已经调用了`isolateSource` 和 `isolateSink`。通过隔离，能防止[代码**冲突（collisions）**](https://en.wikipedia.org/wiki/Collision_%28computer_science%29)，每个组件可以工作得就像当前应用只有这一个组件一样。

每一个 driver 应当定义静态方法 `isolateSource` 和 `isolateSink`。但我们发现，只在 DOM Driver 中，提供了这两个方法，但是，也有一些其他的 driver 用例展示了隔离技术的重要意义。想要学习更多，就请接着阅读 [Drivers](drivers.html)
