---
title: Cycle.js学习笔记5--关于框架设计和抽象
date: 2017-09-16 12:28:23
categories: cyclejs哈根达斯
tags: 笔记
---
因为对Rxjs的好感玩上了Cycle.js，《Cycle.js学习笔记》系列用于记录使用该框架的一些笔记。
本文我们来细细品味一下整个Cyclejs的核心--流，从不一样的角度开始思考吧。
<!--more-->

## Cycle.js基础概念
---
基础知识什么时候恶补都不为晚。

### 人机交互
[官方文档](https://cycle.js.org/dialogue.html)用了这么个高大上的词语形容，“对话抽象”(Dialogue abstraction)、“人机交互”(Human-Computer Interaction)，其实主要讲了以下个东西，容我接地气地讲讲。

首先来看配图：

![image](https://github-imglib-1255459943.cos.ap-chengdu.myqcloud.com/8JS%60Y%7B$TBJ1H%29%7B_~O6DI907.png)

先不管为什么官方文档要放这么一张逼死强迫症的缺个顶部的图片，这图片主要讲了一个轮回：
- 人 => 机：通过操作对机器进行输入，例如从键盘输入内容、用鼠标点击某些元素等
- 机 => 人：机器拿到输入内容，根据程序进行计算，人通过眼睛观察得到输出

其实这里面主要是进行了这样一个抽象：

``` javascript
function main(输入){
    经过一些处理
    return 输出
}
```

我们来看一个简单的场景：操作者需要登录一个页面。

1. 对于操作者--人：
- 输入 => 眼睛获得的图像
- 处理 => 打开浏览器
- 输出 => 输入一个地址，并按Enter

2. 对于机器--电脑：
- 输入 => 浏览器获得地址
- 处理 => 解析地址，发送请求
- 输出 => 返回一个登录页面

3. 对于操作者--人：
- 输入 => 眼睛获得登录页面
- 处理 => 键盘输入帐号密码，登录
- 输出 => 上面的一系列操作便是输出

以上如此轮回的输入和输出，可以简单理解为人机交互。
其实我们在电脑上进行的每一个操作，每一个按键、鼠标移动的每一下，都可以理解为一次输出，也是电脑的一次输入。
而电脑每一个显示的变动或是不变，也都是输出，同时是人通过眼睛的一次输入。

这里面的输入输出有各种的媒介：
电脑的输入包括键盘、鼠标等，输出包括显示器、音响等。
人的输入包括眼睛、耳朵等，输出包括手脚的移动、说话等。

至于这些媒介，我们可以理解为一个接口，或是在Cycle.js里面，可以算是个驱动（Driver）吧，后面章节应该也会讲到。

### 对话抽象
上面所说的人机交互，只是一层简单的抽象，而且也有比较主观的角度。

这里我们尝试再进行一层抽象，不管是人还是机器，都是这样的物体：

``` javascript
function main(输入){
    经过一些处理
    return 输出
}
```

现在我们假设，物体和物体之间的交流通过电、光，所以我们得到这样的物体：

``` javascript
function main(输入){
    光 = 输入.光
    处理光

    电 = 输入.电
    处理电

    // 输出
    return {
        光: 处理后的光
        电: 处理后的电
    }
}
```

这大概是这个世界中所有物体的抽象了。

这时候你可能会有些疑惑，如果每个物体都是这样，那承接这些光和电的传播途径或者方式呢？

到这里，事情变得有意思了，理所当然地，承接这些光和电的便是我们的世界。
而我们的世界又是什么呢？其实也是这样一个抽象的物体：

``` javascript
function 处理器(输入){
    光 = 输入.光
    处理光（传输光）

    电 = 输入.电
    处理电（传输电）

    // 输出
    return {
        光: 处理后的光
        电: 处理后的电
    }
}
创建(处理器, {
    光: 创建光源（驱动器）
    电：创建电源（驱动器）
})
```

说白了，就是世界除了作为一个普通抽象物体，拥有对光和电的处理（例如传播过程减弱等等），同时还有一个特殊的任务：负责创建光和电的驱动。

这个驱动是什么呢？

### 驱动
这个驱动，跟我们平时理解的电脑的驱动有什么区别呢？其实说一样也是一样，说不一样也可以不一样。

其实这里的驱动相当于一些规则，在这里我们理解为对光和电的规则：

``` javascript
function 光驱动器() {
  return 创建流({
    天亮了: 监听器 => {
      创建光
      光.改变 = (增强/减弱) => {
        监听器.通知(增强/减弱)
      }
      光.停止 = (停止) => {
        监听器.通知(停止)
      }
    },
    天黑了: () => {
      光.消失();
    },
  });
}
```

中文的代码是不是看起来好难懂，简单的说，这里面包括一下规则：
1. 光的传播处理（`改变 => 监听 => 通知`）
2. 光的出现消失（`出现 => 开始监听`, `消失 => 关闭监听`）

而所有的抽象物体里，输入和输出都遵守这样的处理。

到这里，我们给这个世界创建了物体，然后规定了光和电的规则，创建世界的时候我们便开始根据设定运作起来。

### 流
现在还剩下最后一个概念：流。

其实流是最好理解的，它就是我们上面所说的光和电。

我们创建一个世界之后：
1. 流的规则设定好，流开始流动；
2. 流的流动，表现为世界对流的输入 => 处理 => 输出；
3. 物体和世界、物体间通过流来进行交流，表现为输入 => 处理 => 输出；
4. 若世界消失，流消失，物体无输入也便无输出，表现为消失。

[捂脸]感觉自己说的也好抽象，不过这是一个挺有意思的角度。不管Cycle.js适不适合用来做项目，但是这样一些新的想法和理念对设计和思路都能拓展空间。

这样的思路拓展到我们的页面中，便有了Cycle.js：

``` javascript
function main(sources) {
  // 流的处理
  
  return {
    DOM: DOM$, // 处理后的DOM流
    router: router$, // 处理后的router流
    HTTP: HTTP$ // 处理后的HTTP流
  };
}

run(main, {
  DOM: makeDOMDriver(), // DOM驱动
  router: makeRouterDriver(), // 路由驱动
  HTTP: makeHTTPDriver() // HTTP驱动
});
```

## 结束语
-----
这节主要发挥本骚年的大脑洞，来进行思想探索。
主要是尝试讲解了一下Cycle.js的设计思路，当然这只是本骚年的理解，或许有些理解不到位，甚至完全颠倒的地方，请多多包涵，和仅供参考。