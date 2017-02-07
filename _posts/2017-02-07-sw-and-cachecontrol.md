---
title: 缓存最佳实践和 max-age 的陷阱
subtitle: 翻译，原文 Jake Archibald - Caching best practices & max-age gotchas
date: 2017-02-07
categories: 笔记
author: Jimmy Q
permalink: cache-best-practices
---

> 又是一篇翻译，因为我对这个确实不太了解。原文地址：[Caching best practices & max-age gotchas](https://jakearchibald.com/2016/caching-best-practices/)又是一篇翻译，因为我对这个确实不太了解。原文地址：

正确的缓存策略会带来巨大的性能提升，节省带宽，节约服务器开销。但是很多网站对缓存考虑不周，造成资源竞争状态导致相互依赖的资源不能同步。

多数缓存的最佳实践可被归类为一下两种模式：

## 模式一：不可（轻易）更改的资源 + 很长的 max-age

<h4 style="background: #444; color: #ddd; font-family: monospace, monospace;">
    Cache-Control: max-age=31536000
</h4>

* 这个URL的资源从不改变，因此……
* 浏览器／CDN可以缓存这个资源一年
* 使用比max-age'年轻的'缓存的资源可以不必咨询服务器

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