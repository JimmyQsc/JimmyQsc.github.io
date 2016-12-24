---
title: Service Worker的生命周期
subtitle: 构建离线应用
date: 2016-11-23
categories: 笔记
author: Jimmy Q
permalink: sw-life-cycle
---

> Note: 翻译自 _The Service Worker Lifecycle  - by Jake Archibald_， Jake Archibald参与了service worker标准的制定，也是该技术的推动者。（下文括号里的内容非作者原文）

service worker的生命周期是它最复杂的部分。如果你不知道它在努力做什么和这么做的优势，你会感到它在跟你对着干。但一旦你知道了它的原理，你就可以给用户提供无缝的，优雅而不突兀的更新。一种同时具备网站应用和原生应用优势的体验。

这是一次深入的探索，但要点会在每个部分开头指出。

## 目的

生命周期的目的是：

* 使离线优先成为可能
* 允许service worker在版本更新时不影响当前版本的运行
* 确保一个网站总是被 **一个** service worker控制（或者没有被控制）
* 确保同时只有一个版本的网站在运行

最后一点是很重要的。没有service worker，用户可能在一个标签中打开你的网站，然后再在另一个标签打开。这可能会导致网站的不同版本同时运行。有时这样做并无大碍，但是当你需要处理数据时，不同版本的同一网站共享着同样的存储空间（你用service worker把数据缓存在本地，通常缓存是按网站来管理的），而它们可能要求用不同的方式来管理这些数据（删除，增加和更新）。这会导致错误甚至数据丢失。

> 注意：用户很不喜欢数据丢失。这会让他们很伤心

## 首次加载的 service worker
概括来说：

* `install`事件是service worker得到的第一个事件，而且只会发生一次
* 传入`installEvent.waitUntil()`的promise对象决定了service worker安装的时长和成功或失败的情况
* service worker不会接收`fetch` 和`push`事件直到它被成功的安装并变为活动状态
* 默认情况下，只有网页本身由service worker得到（由它缓存并从缓存中返回给请求，与之相对的是从网络中得到），它的请求才会被service worker捕获。所以你需要刷新网页才能看到它的效果
* `clients.claim()`可以改变默认值，然后控制没有被控制的网页

看以下的html代码

```html
<!DOCTYPE html>
An image will appear here in 3 seconds:
<script>
  navigator.serviceWorker.register('/sw.js')
    .then(reg => console.log('SW registered!', reg))
    .catch(err => console.log('Boo!', err));

  setTimeout(() => {
    const img = new Image();
    img.src = '/dog.svg';
    document.body.appendChild(img);
  }, 3000);
</script>
```
它做的是注册一个service worker，3秒后添加一个狗的图片。

下面是他的service worker

```javascript
self.addEventListener('install', event => {
  console.log('V1 installing…');

  // 缓存猫猫图片
  event.waitUntil(
    caches.open('static-v1').then(cache => cache.add('/cat.svg'))
  );
});

self.addEventListener('activate', event => {
  console.log('V1 now ready to handle fetches!');
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // 如果请求是同源的并且路径是 '/dog.svg'
  // 则从缓存中拿出猫的图片'/cat.svg'给那个请求
  if (url.origin == location.origin && url.pathname == '/dog.svg') {
    event.respondWith(caches.match('/cat.svg'));
  }
});
```

上面的代码会缓存一个猫的图片，当页面请求`/dog.svg`的时候给它。然而，运行上述代码，你会首先看到狗的图片。刷新页面后你才会看到猫的图片。

> 提示：猫就是比狗可爱。

### 作用域和控制

service worker注册的默认作用域是`./`相对于它自己所在的路径（你可以通过`navigator.serviceWorker.register('/sw.js', { scope: '/'})`第二个参数来明确指定一个scope）。这意味着如果你的service worker的URL是`//example.com/foo/bar.js`它的默认作用域就是`//example.com/foo/`。

我们把页面，workers，和shared workers（一种worker，这东西也有自己的api）称为`clients`。你的service worker只能控制作用域之内的client，一旦client被控制，它发出的fetch请求会被作用域内的service worker捕获，你可以在此时介入。你可以使用`navigator.serviceWorker.controller`方法来判断一个client是否被控制了，这个方法的返回值是`null`或者一个service worker实例。

### 下载，解析和执行

当你调用`.register()`方法时（第一个参数是service worker的URL），service worker会被下载。如果下载失败或者解析（parse）失败或者在首次执行时出现错误（throw error in initial execution），register返回的promise会被reject。service worker会被忽略（abandoned -> redundant）。

chrome开发者工具会把错误显示在console和application tab中的service worker部分。

### 安装

service worker得到的第一个事件是`install`。当service worker开始执行时会立即触发这个事件，每个service worker只触发一次。开发者可以对其进行更新，浏览器认为这个更新是全新的service worker，它会得到一个安装事件（每次加载页面浏览器会检查service worker的更新）。后面会详细讲它的_更新_阶段。

安装事件发生时就是你缓存在控制client之前需要的资源的时候。传入`event,waitUntil()`方法的promise对象会告诉浏览器什么时候安装结束，以及安装是否成功。（fulfilled -> 安装成功，rejected -> 安装失败）。

如果promise rejected，代表安装失败，浏览器会抛弃那个service worker，他不会控制clients。在上面的例子的service worker代码中，这意味着我们可以确保service worker得到fetch事件时（第14行）`cat.svg`会作为页面的依赖出现在缓存中（安装成功意味着安装过程中的缓存文件操作必然成功，因此开发者应该在这个阶段缓存页面的依赖，即页面加载必要的资源）。

### 激活

一旦你的service worker准备好了控制clients并可以处理一些功能性事件例如`push`和`sync`（service worker能做很多强大的事），你会得到一个`activate`事件。但这不代表调用`register()`方法的页面会被立即控制。

当你首次打开上面的例子时，即使`dog.svg`的请求发生在service worker的`activate`事件之后，service worker还无法处理这些请求，你会看到狗的图片。默认规则确保了一致性，如果你的页面不是service worker返回的（用什么中文词来合适的代表serve？）的，页面的请求也不会由它来介入。如果你刷新上面的例子，service worker才会控制页面以及它发出的请求（这时页面本身就是service worker提供的）。页面和图片的请求都会被service worker监听（和介入），你会看到猫的图片。

### `clients.claim`

你可以调用`clients.claim()`方法在service worker激活后立即控制未被控制的clients。

这里有个上面例子的[变种](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/df4cae41fa658c4ec1fa7b0d2de05f8ba6d43c94/)，它会在service worker的激活状态调用`clients.claim`方法。你_应该_第一次加载就能看到猫的图片。我说“应该”是因为这与时间有关。只有当service worker处于活动状态并且`clients.claim`在页面试图下载图片之前生效你才会看到猫的图片。


> Note: 我看很多人把`clients.claim()`放在他们的boilerplate中，我自己很少这么做。这样做只在首次加载时有效果，由于渐进增强（遵循这个原则，开发者不会把service worker当作页面运行的依赖）的功劳，没有service worker的网页照样正常的运行。

## 更新 service worker

概括来说：

<ul>
    <li>下列情况会触发一次更新（浏览器检查service worker的更新）
         <ul>
             <li>访问作用域下的页面</li>
             <li>当`push`和`sync`等功能性事件发生时，除非在之前的24小时之内做过更新检查</li>
             <li>在service worker的URL改变后调用`register()`</li>
         </ul>
    </li>
            <li>你可以设置service worker的http缓存时间（response headers 中的`Cache-Control`），24小时之内浏览器会尊重这个缓存时间（如果你设置的缓存时间为25小时，但在24小时过后上面的三种情况之一发生，浏览器就会更新service worker）。我们（标准制定者）将来会将取消强制的24小时更新行为。多数情况下，你应该把service worker脚本的`max-age`设为0</li>
    <li>你的service worker在更新时，新文件只要跟浏览器已有的文件有一个字节的不同，就会被当作更新。我们会把这样的政策扩展到导入（imported）的模块／脚本</li>
    <li>更新的service worker会和旧的service worker并存，并得到自己的`install`事件</li>
    <li>如果新的worker未被成功下载，或者解析错误，或者在运行时出错，或者在安装阶段不成功，新的worker会被丢弃，旧的会被保留</li>
    <li>一旦新的worker被成功安装，更新的worker会进入等待状态（只有有旧的worker控制clients时才有等待状态），直到旧的worker控制的clients数量为0</li>
    <li>`self.skipWaiting()`会跳过等待，直接让新的worker在安装后进入激活状态</li>

</ul>

 下面我们更改我们的service worker的脚本，使它用马的图片来响应`fetch`

```javascript
const expectedCaches = ['static-v2'];

self.addEventListener('install', event => {
  console.log('V2 installing…');

  // 把马的图片存入新的缓存, static-v2
  event.waitUntil(
    caches.open('static-v2').then(cache => cache.add('/horse.svg'))
  );
});

self.addEventListener('activate', event => {
  // 删除不需要的缓存 static-v1
  event.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (!expectedCaches.includes(key)) {
          return caches.delete(key);
        }
      })
    )).then(() => {
      console.log('V2 now ready to handle fetches!');
    })
  );
});

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  // serve the cat SVG from the cache if the request is
  // same-origin and the path is '/dog.svg'
  if (url.origin == location.origin && url.pathname == '/dog.svg') {
    event.respondWith(caches.match('/horse.svg'));
  }
});
```
> Note: 我对马无感

点击[这里](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v2.html)查看上面代码的demo，你仍然会看到猫的图片，下面是原因：

### 安装

注意到我把缓存的名称从`static-v1`改成了`static-v2`。这意味着我可以重设一个缓存空间，而不必覆盖当前的缓存，旧的service worker还在使用这个缓存。

这种模式创造了版本特定的缓存（每个版本的service worker有自己的缓存空间），就像原生应用把它的可执行文件和它的静态资源一起打包发布版本。当然你也可以设置非版本特定的资源，例如缩略图等。

### 等待

更新的service worker在被成功安装后，会延迟激活，直到旧的service worker不再控制clients。这个状态叫等待状态（waiting state）。浏览器用这种方法来保证同时只有一个service worker在运行。

刷新页面不足以让新的service worker控制页面。这是因为浏览器转到新页面时，当前页面在请求的header被收到之后才会消失，甚至在那时旧的页面还可能会停留因为请求的header里包含了`Content-Disposition`。因为这种__重叠__，当前的worker在刷新时总是控制着页面。

要想得到更新，你得关掉这个标签或者先转到别的网站，然后你再打开这个网站（“重启”这个网站），你就会看到新的worker生效了。

这种模式和许多应用程序的更新模式差不多（比如chrome），它会在后台下载好更新，当它重启后更新才会被安装生效。同时，你可以继续使用当前的版本。然而，这在开发时很不方便（因为你需要做很多实验），但是开发者工具提供了解决方法，这个我在下面会提到。

### 激活

一旦旧的service worker不再生效，更新的版本就会开始控制clients。这时最好做一些旧版本仍在生效时不能做的事，例如迁移数据库和清理缓存。

在上面的例子中，我设置了需要缓存的文件的数组，在`activate`事件中我清除其他缓存（之前版本的）。

> 注意： 用户可能跨版本升级。

如果你给`event.waitUntil()`传入一个promise，它会推迟functional event（`fetch`, `push`, `sync`等等）直到promise resolve。因此当你的fetch事件发生时，激活已经完成了。

> 注意：cache storage API是同源存储（像localstorage， 和indexedDB）。如果你在同源上运行很多网站，当心不要误删除了其他网站的缓存。你可以给你的缓存加一个以网站名区分的独特的前缀。

### 跳过等待阶段

更新worker的等待阶段意味着你同时只能运行一个版本的同个网站，但如果你不需要这项功能，你可以调用`self.skipWaiting()`方法使新的worker提前激活。

这将导致新的worker赶走当前活动的worker，然后在进入等待阶段后立即激活（已在等待阶段则会立即激活）。这不会使你的worker跳过安装阶段。

你在何时调用`skipWaiting()`其实不重要，只要在等待阶段中或者在等待前。在安装阶段调用是通常的做法：

```javascript
self.addEventListener('install', event => {
  self.skipWaiting();

  event.waitUntil(
    // caching etc
  );
});
```

但是你应该让这种行为基于用户的行为，在用户点击某个按钮后，给你的worker发送`postMessage()`。

[这里有个例子](https://cdn.rawgit.com/jakearchibald/80368b84ac1ae8e229fc90b3fe826301/raw/ad55049bee9b11d47f1f7d19a73bf3306d156f43/index-v3.html)使用了`skipWaiting()`。代开后你应该会看到一幅牛的图片。就像`clients.claim()` 这种情况也与时间相关，因此你只有在新的worker安装并激活开始介入fetch等发生在页面加载图片之前才能看到牛的图片。


> 注意：skipWaiting()意味着新的worker可以控制用旧版本的worker加载的页面。这意味着有的网络请求是被旧的worker处理了，而新的worker会处理它控制页面后的请求。这种情况可能会造成破坏，请谨慎使用skipWaiting()。

### 手动更新

我之前提到过，浏览器会在访问页面和事件后自动检查service worker的更新，但你也可以手动触发更新：

```javascript
navigator.serviceWorker.register('/sw.js').then(reg => {
  // sometime later…
  reg.update();
});
```

如果你预料到你的用户可能会长时间使用你的网站而不关闭或重新加载，你可能想要定时使用`update()`方法检查更新。

### 避免更改service worker脚本的地址

如果你读我我的文章[《缓存最佳实践》（Caching best practices & max-age gotchas）](https://jakearchibald.com/2016/caching-best-practices/)，你应该考虑给每个版本的service worker一个独特的URL。__别这么做！__这么做通常是坏实践，尽量在当前URL更新。

设想下面的场景：

1. `index.html`注册了`sw-v1.js`作为它的service worker
2.  `sw-v1.js`缓存并serve`index.html`让它离线优先的工作
3.  你更新了`index.html`好让它注册新的worker：`sw-v2.js`

如果你像上面那么做，用户永远得不到`sw-v2.js`，因为`sw-v1.js`总是会从缓存中拿出旧的`index.html`来响应对其的请求。你让自己陷入了这样的状况，你得在`sw-v1.js`做更新来让用户得到`sw-v2.js`，好蠢。

## 让开发容易

设计service worker的生命周期是以用户为中心的，但它会成为开发过程的阻碍。幸亏有一些开发工具可以帮助你：

### 重载时更新

这会更改service worker的生命周期，使其变得开发友好。每次加载网页都会：

1. 重新下载service worker
2. 把它当作新版本安装，即使它本身没有改变，意味着install事件的代码会运行，缓存会被更新
3.  跳过等待，让新的worker激活
4. 导航到页面

着意味着每次你访问页面（包括刷新）都会得到更新，而不必刷新两次或者关闭标签。


### 跳过等待

如果有个worker正在等待，你可以点击skipWaiting使它立即激活

### 按shift刷新

如果你硬性重新加载页面，它会完全绕过service worker。页面不会被控制。这个功能在标准文档中，所以这个在别的支持service worker的浏览器也有效。

## 处理更新

service worker的设计是为使它成为[可扩展web平台（extensible web）](https://extensiblewebmanifesto.org/)的一部分。核心思想是，作为浏览器开发者，我们不比web开发者更擅长web开发。因此我们不应该提供狭隘的，只解决特定问题的API，而应该让开发者利用浏览器的核心，按自己的方法来做，按更适合自己的用户的方式来做。

因此，为了使更多的开发模式成为可能，service worker的整个更新周期都是可见的：

```javascript
navigator.serviceWorker.register('/sw.js').then(reg => {
  reg.installing; // 正在安装的 worker, 或者 undefined
  reg.waiting; // 处于等待阶段的 worker, 或者 undefined
  reg.active; // 激活状态的 worker, 或者 undefined

  reg.addEventListener('updatefound', () => {
    // 有一个service worker正在安装
    const newWorker = reg.installing;

    newWorker.state;
    // "installing" - install 事件被触发, 但没有完成
    // "installed"  - 安装完成
    // "activating" - activate 事件被触发, 但没有完成
    // "activated"  - 完全激活
    // "redundant"  - 被忽略。安装失败或者被新的worker取代

    newWorker.addEventListener('statechange', () => {
      // newWorker的状态发生改变
    });
  });
});

navigator.serviceWorker.addEventListener('controllerchange', () => {
  // 控制页面的service worker发生改变后会触发这个事件
  // 例如：一个新的service worker跳过了等待阶段变成活跃的worker
});
```

## 总结（非作者原文）

作者在文章开始说了生命周期的四个目的，我们现在看一下这些目的是如何达成的：

### 使离线优先成为可能

开发者在service worker安装阶段缓存页面的依赖，当用户再次导航到页面，它会直接响应请求，返回缓存中的文件。这样使得用户再次加载页面的速度大大提升，也意味着你更新了页面也必须更新一下service worker使更新生效。

此外，除了页面依赖，页面请求的其他数据你也可以用service worker缓存到本地（一旦页面被service worker控制，它能监听到页面发出的任何请求），用户再次打开你的应用，首先显示已缓存的资源，然后进行网络请求。

### 允许service worker在版本更新时不影响当前版本的运行

用户导航到页面，浏览器会检查service worker的更新，这时更新的service worker只会执行安装操作，然后进入等待状态，不打断当前版本的运行。这时开发者可以通知用户有更新，然后让用户决定是否更新。你可以用`skipWaiting()`方法跳过等待，然后刷新页面，让旧的版本消失。

这时，新的worker处于激活状态，我们也只有在这时才能清理之前的缓存，迁移数据库。

### 确保一个网站总是被 **一个** service worker控制（或者没有被控制）

网站不会被多个service worker控制，即使你使用skipWaiting()方法，这也会首先废弃旧的service worker，然后才让新的worker生效。

### 确保同时只有一个版本的网站在运行

事实上，service worker的版本就是网站的版本，由于网站同时只能有一个service worker控制，因此你也只能同时运行一个版本的网站。

同时，不同版本的缓存也不会起冲突，因为worker总是在安装阶段缓存资源，在激活阶段就会清除无用的缓存（开发者自己编程实现）。




