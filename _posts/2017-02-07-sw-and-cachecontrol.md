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

<div class="dialog">

</div>