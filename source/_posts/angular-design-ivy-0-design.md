---
title: Angular框架解读--Ivy编译器整体设计
date: 2021-08-15 13:53:34
categories: Angular源码
tags: 功能设计
---

作为“为大型前端项目”而设计的前端框架，Angular 其实有许多值得参考和学习的设计，本系列主要用于研究这些设计和功能的实现原理。本文围绕 Angular 的核心功能 Ivy 编译器，首先介绍该编译器的整体设计。

<!--more-->

对于前端框架来说，模板编译器（渲染器）属于非常核心的能力了。在 Angular 8.0 中引入了一个新的模板编译器——Ivy 编译器，在这之前 Angular 一直使用 View Engine 来编译模板。

## Ivy 编译器能力

编译器的用途，基本上是将开发者编写的代码，编译成可在浏览器中运行的代码。使用了编译器之后，前端框架就可以定义很多自身的语法，在编译过程可以给代码增加一些性能优化、安全检测等功能。对于 Angular 来说，编译器还需要支持将开发者代码编译成 AOT 和 JIT 两种。

Angular 重构编译器并将之命名为 Ivy 编译器，这对于 Angular 框架来说有着非常重要的意义，有点类似于 React 重构 Fiber。

### Ivy 新特性

我们先来看看 Ivy 编译器的一些特性，包括但不限于以下的内容：

- 🚀 缩短构建时间（增加增量编译）
- 🔥 达到更好的构建大小（生成的代码和 Tree-shaking 更兼容），有效地降低代码包大小
- 🔓 解锁新的潜在功能（元编程或更高级别的组件，支持组件的延迟加载，支持不基于 zone.js 的新变更检测系统，等等）

前面章节中我们有介绍 Angular 的[元编程](https://godbasin.github.io/2021/03/27/angular-design-metadata/)、组件和模块之间的关系、[zone.js 中的设计](https://godbasin.github.io/2021/05/01/angular-design-zonejs/)和[引入](https://godbasin.github.io/2021/05/30/angular-design-zone-ngzone/)等内容，其中不少能力和设计都无法与 Ivy 编译器的设计和引入脱离关系。比如，[Angular 依赖设计](https://godbasin.github.io/2021/07/11/angular-design-di-2-hierarchical-di/)中由于延迟模块引入的 bug，前面我们说过 Angular 中的依赖注入通过将注入器分为元素注入器和模块注入器，而在 Ivy 编译器中，使用了支持到组件级别的延迟加载（Node ），最终解决了延迟模块重复加载的问题。

今天我们先来了解一下 Ivy 编译器的整体设计，后面会再分具体的章节来详细介绍内部的一些源码实现。

## Ivy 架构设计

在 Angular 中，开发者编写的代码大多数为 Typescript 代码，其中还包括了许多 Angular 提供的 API 和语法糖，因此 Angular 需要通过语法解析转换为 AST，并根据 AST 编译成最终可以跑在浏览器中的代码，这便是 Ivy 编译器需要实现的核心能力。

Ivy 编译器主要包括两部分：

1. `ngtsc`是一个 Typescript-to-Javascript 编译器，它将 Angular 装饰器化为静态属性。它是一个最小包装器，包裹在`tsc`之外，而`tsc`中则包含一系列的 Angular 变换。
2. `ngcc`主要负责处理来自 NPM 的代码并生成等效的 Ivy 版本，就像使用`ngtsc`编译代码一样。

### 模板编译

在 Ivy 编译器中使用`TemplateCompiler`来编译模板，该过程中会执行以下操作：

1. 标记模板。
2. 将标记内容解析为 HTML AST。
3. 将 HTML AST 转换为 Angular 模板 AST。
4. 将 Angular 模板 AST 转换为模板函数。

Angular Template AST 转换和注释的 HTML AST 版本时，会执行以下操作：

1. 将 Angular 模板语法快捷方式（例如`*ngFor`和`[name]`）转换为其规范版本（和`bind-name`）。
2. 收集引用（`#`属性）和变量（`let-`属性）。
3. 使用收集的变量和引用，解析并转换绑定表达式 AST 中的绑定表达式。

除了以上操作之外，该过程还会生成详尽的选择器目标列表，包括任何组件、指令或管道的选择器的潜在目标。确定组件包含它所依赖的组件、指令和管道的列表，可在运行时知道将哪些组件和指令应用于元素以及绑定表达式引用了哪些管道。从而`TemplateCompiler`可以从字符串生成模板函数，而无需附加信息。

确定此列表的过程称为引用反转，因为它将从模块（包含依赖项）到组件的链接反转为从组件到其依赖项的链接。然后，程序只需要包含呈现的初始组件所依赖的类型以及这些依赖项所需的任何类型。除此之外，还解决了 tree-shaking 的问题。

### Typescript 解析器

要实现 AST 的解析和转换，离不开解析器。对于 Typescript 代码来说，编译器的整体流程为：

```
                                                                |------------|
                           |----------------------------------> | TypeScript |
                           |                                    |   .d.ts    |
                           |                                    |------------|
                           |
|------------|          |-----|               |-----|           |------------|
| TypeScript | -parse-> | AST | ->transform-> | AST | ->print-> | JavaScript |
|   source   |    |     |-----|       |       |-----|           |   source   |
|------------|    |        |          |                         |------------|
                  |    type-check     |
                  |        |          |
                  |        v          |
                  |    |--------|     |
                  |--> | errors | <---|
                       |--------|
```

其中，解析步骤是传统的递归下降解析器，经过增强以支持增量解析，该解析器发出抽象语法树（AST）。转换步骤是一组 AST 到 AST 的转换，这些转换执行各种任务，例如删除类型声明，将模块和类声明降低到 ES5，将`async`方法转换为状态机等。

### 编译器设计

前面我们提到 Ivy 支持增量编译，从而达到缩短构建时间的效果。增量编译的预期是当一个库已经被编译时，我们就不必每次都重新编译它，而是根据变更的部分进行重新编译。这看起来比较简单，实际上它对编译器提供了不小的挑战，因为组件的生成代码可能会使用另一个组件的内部细节。

从广义上讲，Ivy 模型是将 Angular 装饰器编译为类上的静态属性，包括：

- 组件编译（`ViewCompiler`和样式编译器）：编译`@Component` => `ɵɵdefineComponent`
- 管道编译`PipeCompiler`：编译`@Pipe`=>`ɵɵdefinePipe`
- 指令编译`DirectiveCompiler`：编译`@Directive`=>`ɵɵdefineDirective`
- 可注入编译`InjectableCompiler`：编译`@Injectable`=>`ɵɵdefineInjectable`
- 模块编译`NgModuleCompiler`：编译`@NgModule`=>`ɵɵdefineInjector`（`ɵɵdefineNgModule`仅在 JIT 中）

这些操作必须在没有全局程序数据的情况下进行，并且在大多数情况下，仅在具有单个装饰器数据的情况下进行。

因此，Ivy 编译器不得依赖未直接传递给它的任何输入（例如，它不得扫描源或元数据中的其他数据）。该限制很重要，原因有两个：

1. 由于可以看到编译器的所有输入，因此它有助于强制执行 Ivy 局部性原则。
2. 它可以防止在`--watch`模式下进行错误的构建，因为文件之间的依赖关系很容易跟踪。

所以在 Ivy 中，每个将单个装饰器转换为静态字段的“编译器”都将充当“纯函数”。给定有关特定类型和装饰器的输入元数据，它将生成一个对象，该对象描述要添加到该类型的字段，以及该字段的初始化值（采用 AST 格式）。

举个例子，`@Component`编译器的输入包括以下内容：

- 对组件类的引用
- 组件的模板和样式资源
- 组件的选择器
- 组件所属模块的选择器映射

### Ivy 编译模型

在 Angular 中，实例化组件、创建 DOM 节点以及运行变更检测，以上的逻辑被实现为一个原子单位，被称为“Angular 解释器”。编译器仅生成有关其模板中定义的组件和元素的元数据。

在旧版 View Engine 中，编译过程为：

![](https://github-imglib-1255459943.cos.ap-chengdu.myqcloud.com/angular-ivy-1-1.png)

`<span>My name is {{name}}</span>`这句模板代码，编译后的代码大概长这个样子：

```js
viewDef(0,[
    elementDef(0,null,null,1,'span',...),
    textDef(null,['My name is ',...])
]
```

而在 Ivy 编译器中，编译后的代码大概长这个样子：

```js
// create mode
if (rf & RenderFlags.Create) {
  elementStart(0, "span");
  text(1);
  elementEnd();
}
// update mode
if (rf & RenderFlags.Update) {
  textBinding(1, interpolation1("My name is", ctx.name));
}
```

在 Ivy 编译器中，编译过程为：

![](https://github-imglib-1255459943.cos.ap-chengdu.myqcloud.com/angular-ivy-1-2.png)

在 View Engine 中，组件定义（模板数据）独立于组件类而位于其自己的文件中。而在 Ivy 编译器中，组件定义将通过静态字段附加到组件类，编译期间不会创建单独的文件。

## 总结
今天大致介绍了 Angular 中 Ivy 编译器的整体设计，其实 Angular 本身就对 Ivy 编译器的整体架构提供了很详细的说明，本文很多内容也都参考来自这些内容，建议大家可以都去看看 [DESIGN DOC(Ivy): Compiler Architecture](https://github.com/angular/angular/blob/master/packages/compiler/design/architecture.md)。

Ivy 编译器作为 Angular 的核心能力，并不是一篇文章足以概括完毕。本文也并未介绍`ngtsc`的编译流程、资源加载等内容，也并未开始结合 Angular 中的源码一起研究其实现。这些我后续会尝试一点一点地挖掘，希望能从中学到架构文档以外更多的知识，我也会尝试将自己的学习过程记录下来，继续分享给对 Angular 感兴趣的你们~~

### 参考

- [DESIGN DOC(Ivy): Compiler Architecture](https://github.com/angular/angular/blob/master/packages/compiler/design/architecture.md)
- [Under the hood of the Angular Compatibility Compiler (ngcc)](https://indepth.dev/posts/1259/angular-compatability-compiler)
- [What is Angular Ivy?](https://blog.ninja-squad.com/2019/05/07/what-is-angular-ivy/)
- [All you need to know about Ivy, The new Angular engine!](https://medium.com/angular-in-depth/all-you-need-to-know-about-ivy-the-new-angular-engine-9cde471f42cf)
- [Ivy engine in Angular: first in-depth look at compilation, runtime and change detection](https://indepth.dev/posts/1062/ivy-engine-in-angular-first-in-depth-look-at-compilation-runtime-and-change-detection)
- [Everything you need to know about change detection in Angular](https://indepth.dev/posts/1053/everything-you-need-to-know-about-change-detection-in-angular)
