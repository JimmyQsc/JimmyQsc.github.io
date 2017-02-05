---
title: 任务，微任务，队列和调度
subtitle: 翻译，原文 Jake Archibald - Tasks, microtasks, queues and schedules
date: 2016-01-23
categories: 笔记
author: Jimmy Q
permalink: t-m-q-s
---

> 翻译自 _[Tasks, microtasks, queues and schedules - Jake Archibald](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)_
坐好了，这篇有很多规范。

看下面的代码：

```javascript
    console.log('script start');

    setTimeout(function() {
    console.log('setTimeout');
    }, 0);

    Promise.resolve().then(function() {
    console.log('promise1');
    }).then(function() {
    console.log('promise2');
    });

    console.log('script end');
```
输出的顺序应该是怎样的？请在console中试试

## Try it

正确的顺序应该是：`script start`, `script end`, `promise1`, `promise2`, `setTimeout`, 但是浏览器的表现大不相同。

微软的edge， Firefox 40，iOS Safari 和桌面 Safari 8.0.8 输出 `setTimeout` 然后输出 `promise1` 和 `promise2` ，尽管这看起来像是race condition。这仍是非常奇怪的表现，因为 Firefox 39 和 Safari 8.0.7 通常会得出正确的结果。

## Why this happens

要理解这个问题你首先应该知道event loop怎样处理tasks和microtasks。这个刚开始可能理解起来很费劲，深呼吸……

每个‘线程‘都有自己的event loop，所以每个web worker都有自己的event loop，这样它才可以独立执行。而所有同源的窗口共享一个event loop，这样他们就能进行同步的通信。event loop无阻断的运行，执行任何入队的task。一个event  loop有多个任务源（task source），这保证了同一任务源的任务的执行顺序（像indexedDB定义了自己的事件源）。但是浏览器有权利选择从哪个事件源选取任务去执行。这样浏览器就可以对一些如用户输入等对性能要求高的任务一些优先级。先别走，再往下看……

事件是被一个个安排好的，好让浏览器把他们放到JS/DOM中并保证这些动作按顺序执行。在任务（task）执行的间隙，浏览器_可能_会渲染页面更新。从鼠标点击到事件回调要求安排一个任务，就如同解析html，和刚才的例子 - `setTimeout`。

`setTimeout`等待给定的时间然后为它的回调安排一个新任务。这就是为什么`setTimeout`会在`script end`后输出，因为输出`script end`是第一个任务的一部分，而输出`setTimeout`是另一个任务了。对，我们快理清上面的问题了，但我需要你保持坚强，来看下面的文字……

microtasks通常被安排给需要在当前代码执行完毕后立即进行的，例如对一些动作作出反应，或者做一些异步的操作并避免新建一个新任务（task）那么麻烦。浏览器会在两种情况下开始处理微任务（microtasks）队列，一是回调结束后，没有别的代码正在执行，二是每个任务（task）结束之后。任何在微任务执行中入队的微任务会排在队尾，同样被处理。微任务包括了[MutationObserver](https://developer.mozilla.org/en/docs/Web/API/MutationObserver)的回调和上面例子中的promise回调。

一旦一个promise到了settle状态（resolve或者reject），或者它早已被settle，会为它对应的回调安排一个微任务队列。这保证了promise回调是异步的即便promise早已被settle。因此当在一个settle的promise后调用`then(yey, nay)`会立即创建微任务。这就是为什么`promise1`和`promise2`会在`script end`后输出，因为微任务被处理前，当前运行的代码必须执行完。`promise1`和`promise2`会在`setTimeout`前输出，因为微任务总是在下一个任务之前被处理。

所以，我们一步步来：[这里看原文](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

### 为什么有的浏览器会出现不同结果？

有些浏览器的输出顺序是：`script start`, `script end`, `setTimeout`, `promise1`, `promise2`。他们在`setTimeout`后运行promise。很可能这些浏览器把promise的回调当作新任务而非微任务。

这个是可原谅的分歧，因为promise来自ECMAScript而不是html。ECMAScript有一个“jobs“的概念，跟微任务（microtasks）很相似，但两者的关系并不明确，除了在[vague mailing list discussions](https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16)有一些讨论。尽管如此，大致的共识是promise回调应该是微任务队列中的，这有很多优点。

把promise当作任务会导致性能问题，因为回调函数可能会因为渲染等任务（task）执行时会出现的的东西产生不必要的延迟。这也会在因为与别的任务交互而造成不确定性，

## 如何分辨tasks和microtasks

如果你想测试的话，用上面的例子，看他们输出顺序相对于promise和setTimeout的情况，但这样依赖于浏览器有正确的实现。

最准确的方式时看规范文档，例如，[step 14 of setTimeout](https://html.spec.whatwg.org/multipage/webappapis.html#timer-initialisation-steps)提到`setTimeout`会创建任务，而[step 5 of queuing a mutation record](https://dom.spec.whatwg.org/#queue-a-mutation-record)提到queue-a-mutation-record会创建微任务。

像我提到的，在ECMAScript标准中，把microtasks叫做jobs。在[step 8.a of PerformPromiseThen](http://www.ecma-international.org/ecma-262/6.0/#sec-performpromisethen)中提到`EnqueueJob`应当创建一个微任务。

现在我们来看一些更复杂的例子，我相信你准备好了，让我们开始吧……

## Level 1 bossfight

在写这篇博客前，我搞错过这个问题，先看html

```html
<div class="outer">
  <div class="inner"></div>
</div>
```
给你下面的js，当我点击`div.inner`时会输出什么？

```javascript
// 拿到元素
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// 监听outer的属性变化
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// …which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```
先自己试一下吧。

## 不同浏览器中测试

## 谁是对的？

Chrome是对的，正确顺序是 click, promise, mutate, click, promise, mutate, timeout, timeout. 





