---
title: 缓存最佳实践和 max-age 的陷阱
subtitle: 翻译，原文 Jake Archibald - Caching best practices & max-age gotchas
date: 2017-02-07
categories: 笔记
author: Jimmy Q
permalink: cache-best-practices
---

> 我发现这篇很有用。原文地址：[Caching best practices & max-age gotchas](https://jakearchibald.com/2016/caching-best-practices/)。

正确的缓存策略会带来巨大的性能提升，节省带宽，节约服务器开销。但是很多网站对缓存考虑不周，造成资源竞争状态导致相互依赖的资源不能同步。

多数缓存的最佳实践可被归类为一下两种模式：

## 模式一：内容不变 + 很长的 max-age

<h5 style="background: #444; color: #ddd; font-family: monospace, monospace;">
    Cache-Control: max-age=31536000
</h5>

* 这个URL的资源从不改变，因此……
* 浏览器／CDN可以缓存这个资源一年
* 使用比max-age'年轻的'缓存的资源可以不必咨询服务器

<div class="chat">
  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要 <span class="chat-nowrap">"/script-v1.js"</span>, <span class="chat-nowrap">"/styles-v1.css"</span> 和 <span class="chat-nowrap">"/cats-v1.jpg"</span>
    <span class="time">10:24</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我这儿没有，Server 你呢?
    <span class="time">10:24</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    有，给你。 Btw 缓存：这些你可以用一年。
    <span class="time">10:25</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    谢谢!
    <span class="time">10:25</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    好开心!
    <span class="time">10:25</span>
  </p>

  <p class="chat-direction">第二天</p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要 <span class="chat-nowrap">"/script-<strong>v2</strong>.js"</span>, <span class="chat-nowrap">"/styles-<strong>v2</strong>.css"</span> 和 <span class="chat-nowrap">"/cats-v1.jpg"</span>
    <span class="time">08:14</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我有"/cats-v1.jpg"，给你。其他的我没有，Server?
    <span class="time">08:14</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    当然，这是新的css &amp; JS. Btw 缓存：这些你也可以用一年。
    <span class="time">08:15</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    太棒了!
    <span class="time">08:15</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    谢谢!
    <span class="time">08:15</span>
  </p>

  <p class="chat-direction">然后</p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我好久没用 <span class="chat-nowrap">"/script-v1.js"</span> &amp; <span class="chat-nowrap">"/styles-v1.css"</span> 了。我要把他们删了。
    <span class="time">12:32</span>
  </p>
</div>


这种模式中，你从不更改某个URL的资源，你更改URL：

```html
<script src="/script-f93bca2c.js"></script>
<link rel="stylesheet" href="/styles-a837cb1e.css">
<img src="/cats-0e9a2ef4.jpg" alt="…">
```

每个URL都有一部分会随内容而改变。可能是版本号，更改时间或者内容的hash值。

多数后端框架都提供工具来做这个，有node工具做同样的事，例如gulp-rev。

然而这种模式不适用于文章和博客。它们的URL不能设置版本它们的内容必须可变。

## 模式二：可变内容，总是服务器验证

<h5 style="background: #444; color: #ddd; font-family: monospace, monospace;">
    Cache-Control: no-cache
</h5>

* 这个URL的资源内容可能会改变，因此……
* 任何本地缓存的资源都不被信任，除非server验证

<div class="chat">
  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要这几个资源： <span class="chat-nowrap">"/about/"</span> and <span class="chat-nowrap">"/sw.js"</span>
    <span class="time">11:32</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我这儿没有。server 你有吗？
    <span class="time">11:32</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    我有，给你。缓存: 你可以缓存这些资源, 但用之前问我一下。
    <span class="time">11:33</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    好的。
    <span class="time">11:33</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    谢谢
    <span class="time">11:33</span>
  </p>

  <p class="chat-direction">第二天</p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要这些资源： <span class="chat-nowrap">"/about/"</span> and <span class="chat-nowrap">"/sw.js"</span> again
    <span class="time">09:46</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    等一下。 Server: 我可以给网页这些资源嘛？ 我这儿的 <span class="chat-nowrap">"/about/"</span> 最后一次更新是星期一, <span class="chat-nowrap">"/sw.js"</span> 是昨天的。
    <span class="time">09:46</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    <span class="chat-nowrap">"/sw.js"</span> 还没有改变…
    <span class="time">09:47</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    Cool. 网页: 给你 <span class="chat-nowrap">"/sw.js"</span>。
    <span class="time">09:47</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    …但是 <span class="chat-nowrap">"/about/"</span> 变了, 给你新版本。 缓存: 像以前一样，缓存这些资源，但是用之前要问我一下。
    <span class="time">09:47</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    明白!
    <span class="time">09:47</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Excellent!
    <span class="time">09:47</span>
  </p>
</div>

__注意__：`no-cache` 的意思不是‘不要缓存’，而是必须与server重新验证后才能使用缓存的资源。`no-store`告诉浏览器不要缓存。而且`must-revalidate`的意思
不是‘必须重新验证’，它的意思是本地资源如果比提供的`max-age`年轻，就可以使用，否则重新验证。

这种模式中，可以为响应头添加`Etag`或者`Last-Modified`时间。下次客户端请求资源时，会分别通过`If-None-Match`或者`If-Modified-Since`告诉server本地资源的内容，
让server响应“就用你本地的版本吧，它是最新的”，也就是`HTTP 304`。

如果无法发送`ETag`/`Last-Modified`，服务器总是会发送全部内容。

这种模式总是会发送http请求，所以它不如第一种模式，完全越过的网络请求。

经常见到，被模式一的基础设施要求和模式二的网络请求造成的性能问题所困扰的开发者，采用了一种中间方案：资源内容可变，并设置一个相当小的`max-age`。这是一种糟糕的折衷。

## 为可变资源设置 max-age 通常是错误的

……遗憾的事，这种做法很常见，比如GitHub pages。

想象：

* `/article/`
* `/styles.css`
* `/script.js`

都有这样的响应头：

<h5 style="background: #444; color: #ddd; font-family: monospace, monospace;">
    Cache-Control: must-revalidate, max-age=600
</h5>

* URL对应的资源内容可变
* 如果浏览器缓存的资源不超过10分钟，使用可以不进行网络请求
* 否则进行网络请求，如果可以，使用`If-Modified-Since`或者`If-None-Match`

<div class="chat">
  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要 <span class="chat-nowrap">"/article/"</span>, <span class="chat-nowrap">"/script.js"</span> and <span class="chat-nowrap">"/styles.css"</span>
    <span class="time">10:21</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我这儿没有，server？
    <span class="time">10:21</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    没问题，给你。 Btw 缓存: 这些可以用10分钟。
    <span class="time">10:22</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    了解!
    <span class="time">10:22</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    谢谢!
    <span class="time">10:22</span>
  </p>

  <p class="chat-direction">6 分钟后</p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    Hey, 我需要 <span class="chat-nowrap">"/article/"</span>, <span class="chat-nowrap">"/script.js"</span> 和 <span class="chat-nowrap">"/styles.css"</span> again
    <span class="time">10:28</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    我的天，很抱歉，这几个被我丢了 <span class="chat-nowrap">"/styles.css"</span>, 但我有其他的几个， Server: 你给我这个吧 <span class="chat-nowrap">"/styles.css"</span>?
    <span class="time">10:28</span>
  </p>

  <p class="chat-item server-chat">
    <span class="author">Server<span>:</span></span>
    好的, 这个文件的内容变了，这个文件还可以用10分钟
    <span class="time">10:29</span>
  </p>

  <p class="chat-item cache-chat">
    <span class="author">缓存<span>:</span></span>
    好的。
    <span class="time">10:29</span>
  </p>

  <p class="chat-item page-chat">
    <span class="author">网页<span>:</span></span>
    谢谢! 等等! 页面坏了!! 怎么回事?
    <span class="time">10:29</span>
  </p>
</div>

这种模式在测试时可能不会发现问题，但在真实访问场景中会出现问题，并且很难追踪问题。在上面的例子中，服务器更新了html，js和css，但页面得到的是缓存中旧的html和js，和server返回的
更新的css。版本不匹配会破坏页面。

通常我们在对html做重大改变后也会改变css来反映页面结构的变化，同时更新js。因此这些资源是相互依赖的，但缓存头信息不能反映这种关系。用户可能会得到某些文件的新版本和另一些文件的旧版本。

`max-age`是相对于响应时间的，因此如果上述资源是同一次页面访问时请求的，他们的过期时间大概是相同的。但也有很小的可能用户在它们过期时间的间隙请求资源，造成资源不匹配。如果你有些页面不包括
js，或者包括了不同的css，这些资源的过期时间就会变的不同步。更糟糕的是，某些浏览器缓存不可用的情况时有发生，而浏览器并不知道这些资源是相互依赖的。所以浏览器会无忧无虑的删除一些缓存而保留另一些
。以上各种情况叠加在一起，不同资源版本不匹配的问题就变得很常见了。

对用户来说，可能会造成布局错乱和／或功能不可用。小bug或者完全不可用都有可能。

庆幸的是，用户有办法修正这种错误

### 刷新网页有时会修复这个问题

如果网页是通过刷新得到的，浏览器总是会与server进行重新验证，忽略`max-age`，按刷新按钮会修复所有问题。当然，强迫用户做这种事，会让用户觉得你的网站不稳定。

### Service worker 可能会延长这些bug的寿命

假设你有下面的service worker

```javascript
const version = '2';

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(`static-${version}`)
      .then(cache => cache.addAll([
        '/styles.css',
        '/script.js'
      ]))
  );
});

self.addEventListener('activate', event => {
  // …删除旧缓存…
});

self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});

```

这个service worker会：

* 提前缓存好样式和脚本
* serve 缓存中的资源或者请求网络

如果我们改变了样式／脚本我们会改变版本号以更新service worker。然而，由于`addAll`是从HTTP缓存中获取资源（跟别的网络请求一样），我们可能会遇到`max-age`竞速的
情况从而缓存版本不匹配的样式和脚本。

一旦他们被缓存，你就完了，service worker会一直提供版本不兼容的样式和脚本，直到下次更新 - 而下次更细也可能遇到同样的情况。

你可以通过设置，越过http缓存：

```javascript
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(`static-${version}`)
      .then(cache => cache.addAll([
        new Request('/styles.css', { cache: 'no-cache' }),
        new Request('/script.js', { cache: 'no-cache' })
      ]))
  );
});

```

不幸的是缓存设置在chrome和opera中还不可用，最近才被教导Firefox nightly中，但你也可以自己做：

```javascript
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(`static-${version}`)
      .then(cache => Promise.all(
        [
          '/styles.css',
          '/script.js'
        ].map(url => {
          // 为资源添加随机的query string强制下载资源
          return fetch(`${url}?${Math.random()}`).then(response => {
            // fail on 404, 500 etc
            if (!response.ok) throw Error('Not ok');
            return cache.put(url, response);
          })
        })
      ))
  );
});
```

上面的代码中我用了一个随机数作为query string来销毁缓存，你也可以更进一步使用构建工具来完成这个任务。这个有点像用javascript来实现模式一，但只对service worker有效，
而不是所有的浏览器和cdn。

## service worker 和 HTTP 缓存可以和睦共处，别让它们打架！

如你所见，你可以在service worker中提升糟糕的缓存，但你最好还是从根本上解决问题。正确的缓存策略会让你在service worker中更得心应手，也会让不支持service worker的
浏览器受益，同时发挥cdn的最大潜力。

正确的缓存策略意味着你可以极大的提高service worker更新的效率：

```javascript
const version = '23';

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(`static-${version}`)
      .then(cache => cache.addAll([
        '/',
        '/script-f93bca2c.js',
        '/styles-a837cb1e.css',
        '/cats-0e9a2ef4.jpg'
      ]))
  );
});
```

这里，我会用模式二（server重新验证）来缓存根页面（index.html），其他资源用模式一。每次service worker更新都会引发请求根页面，而其他资源只有在URL改变后才会被下载
。这种方式节约了带宽提升了性能，不论你是从上一个版本升级还是间隔了10个版本。

这种升级方式比原生应用好了一万倍，因为即使是很小的改变，也必须下载全部文件来升级。而我们可以用相对很少的流量来更新web应用。

service worker是一种增强，而不是用来做折衷的，所以不要与缓存打架。

## 谨慎的使用，max-age和可变内容也有好处

为可变内容设置`max-age`通常是错误的，但也不总是。比如，在一个博客系统中，为文章页设置了`max-age`为三分钟。因为文章页的依赖（样式，脚本和图片）都使用模式一，而
依赖于此的资源也不使用和文章页相同的缓存头，竞速状态不是个问题。

这意味着，假如博主写了篇好的博客，最迟3分钟后，读者就会看到更新。

这种模式不应该大规模的使用，如果我在一篇文章中添加了一个板块，并且将其引用到了其他文章，我就制造了一种竞速状态。用户可能会点击链接而进入没有那个板块的版本。要
避免这种情况，我必须更新目前的文章，在cdn中删除缓存，等待三分钟，然后再在另一篇文章中添加链接。是的，你必须小心对待这种模式。

正确的使用缓存，是会极大的优化性能，节约带宽。坚持使用模式一或者模式二。谨慎的为可变资源添加`max-age`，确保这些资源的依赖和依赖于此的资源不使用这种模式。