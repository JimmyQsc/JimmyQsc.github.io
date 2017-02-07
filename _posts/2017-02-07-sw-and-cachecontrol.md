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

## 模式一：不可（轻易）更改的资源 + 很长的 max-age

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

## 模式二：可变内容，总是服务器验证。

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

