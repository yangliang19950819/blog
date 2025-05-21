---
title: service worker
cover: /img/serviceworker-z.webp
---

### service worker 概念和特点

- Service workers 本质上充当 Web 应用程序、浏览器与网络（可用时）之间的代理服务器。这个 API 旨在创建有效的离线体验，它会拦截网络请求并根据网络是否可用采取来适当的动作、更新来自服务器的的资源。它还提供入口以推送通知和访问后台同步 API。

- Service worker 运行在 worker 上下文，因此它不能访问 DOM。相对于驱动应用的主 JavaScript 线程，它运行在其他线程中，所以不会造成阻塞。它设计为完全异步，同步 API（如 XHR 和 localStorage (en-US)）不能在 service worker 中使用。

- 出于安全考量，Service workers 只能由 HTTPS 承载，毕竟修改网络请求的能力暴露给中间人攻击会非常危险。在 Firefox 浏览器的用户隐私模式，Service Worker 不可用。

### service worker 实践

- index.html 浏览器请求的页面，判断是否支持 service worker，注册 service worker，在 service worker 准备好后可以互相通信

```html
<!DOCTYPE html>
<html lang="zh-cn">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>service worker</title>
  </head>
  <body>
    <script>
      if ("serviceWorker" in navigator && navigator.serviceWorker) {
        // 需要判断是否支持service worker
        navigator.serviceWorker
          .register("/webs.js")
          .then((reg) => {
            //  注册service worker '/webs.js'就是service worker编写的文件
            console.log(reg, "🧬");
          })
          .catch((err) => {
            console.error(err);
          });
        navigator.serviceWorker?.ready // 当service worker 准备好后，向service worker线程发送消息
          .then(() => {
            navigator.serviceWorker.controller?.postMessage({
              msg: "client service worker",
            });
          });
        navigator.serviceWorker.addEventListener("message", function (event) {
          // 监听service worker 发送来的消息
          console.log(event, "🇫🇯");
        });
      }
    </script>
  </body>
</html>
```

- webs.js 负责缓存文件 和 拦截 fetch 事件 以下是 mdn 对每个事件的定义
- ExtendableEvent.waitUntil() 方法告诉事件分发器该事件仍在进行。这个方法也可以用于检测进行的任务是否成功。在服务工作线程中，这个方法告诉浏览器事件一直进行，直至 promise 解决，浏览器不应该在事件中的异步操作完成之前终止服务工作线程。
- 服务工作线程（service workers）中的 install (en-US) 事件使用 waitUntil() 来将服务工作线程保持在 installing (en-US) 阶段。如果传入 waitUntil() 的 promise 被拒绝，则将此次安装视为失败，丢弃这个服务工作线程。这主要用于确保在服务工作线程安装以前，所有依赖的核心缓存都已经成功载入。
- 服务工作线程（service workers）中的 activate (en-US) 事件使用 waitUntil() 来延迟函数事件，如 fetch 和 push，直至传入 waitUntil() 的 promise 被解决。这让服务工作线程有时间更新数据库架构（database schema）和删除过时缓存（caches），让其他事件能在一个完成更新的状态下进行。
- waitUntil() 方法最初必须在事件回调里调用，在此之后，方法可以被调用多次，直至所有传入的 promise 被解决。

- CacheStorage 接口的 open() 方法返回一个 resolve 为匹配 cacheName 的 Cache 对象的 Promise .注意:如果指定的 Cache 不存在，则使用该 cacheName 创建一个新的 cache，并返回一个 resolve 为该新 Cache 对象的 Promise.
- Cache 接口的 delete() 方法查询 request 为 key 的 Cache 条目，如果找到，则删除该 Cache 条目并返回 resolve 为 true 的 Promise 。 如果没有找到，则返回 resolve 为 false 的 Promise 。

- FetchEvent 接口的 respondWith() 方法旨在包裹代码，这些代码为来自受控页面的 request 生成自定义的 response。这些代码通过返回一个 Response 、 network error 或者 Fetch 的方式 resolve。
- Response 接口的 clone() 方法创建了一个响应对象的克隆，这个对象在所有方面都是相同的，但是存储在一个不同的变量中。
- Cache 接口的 put() 方法 允许将键/值对添加到当前的 Cache 对象中.

- Clients 接口的 matchAll() 方法返回 service worker Client 对象列表的 Promise . 包含 options 参数以返回域与关联的 service worker 的域相同所有 service worker 的 clients. 如果未包含 options，该方法仅返回由 service worker 控制的 service worker clients.

```js
const cacheName = "v1"; // 缓存的版本号

// const cacheAssets = ['index.html','main.js']  // 手动添加你需要缓存的文件名称

// self.addEventListener('install', function(event) {    // install声明周期
//   event.waitUntil(caches.open(cacheName).then(cache => {
//     cache.addAll(cacheAssets)
//   }).then(() => {self.skipWaiting()})
//   )
// });

self.addEventListener("activate", function (event) {
  // activate 生命周期
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      // 拿到所有缓存 删除匹配不是当前cacheName 版本的缓存
      return Promise.all(
        cacheNames.map((cache) => {
          if (cache !== cacheName) {
            return caches.delete(cache);
          }
        })
      );
    })
  );
});
self.addEventListener("fetch", (e) => {
  // 拦截fetch 自定义返回response
  e.respondWith(
    fetch(e.request)
      .then((res) => {
        const resClone = res.clone(); // clone response
        caches
          .open(cacheName) // 新建cacheStorage 对象
          .then((cache) => {
            if (e.request.url.indexOf(self.location.host) > -1) {
              // 判断url是否同源
              cache.put(e.request.url, resClone); // 将键值对 存入cache对象
            }
          });
        return res;
      })
      .catch(() => caches.match(e.request).then((res) => res))
  );
});
self.addEventListener("message", (event) => {
  self.clients.matchAll().then(function (clients) {
    // 向所有client/windowClient对象发送消息
    clients.forEach(function (client) {
      client.postMessage("serve service worker");
    });
  });
});
```

### 本地调试结果

- 前端开发常用 vscode 插件中下载 live server 插件，直接右键 index.html open with live server，可以启动一个本地服务器
- 进入到页面就可以看到 打开 chrome 控制台，点击 Application 中的 service Workers serviceWorker 的 status 是 activated and is running
- 将页面调成 offline 断网状态刷新页面依旧可以正常运行
  ![avatar](/img/swa.webp)
- cache storage 中可看到按你自己版本号 储存的 cache
  ![avatar](/img/cache.webp)

- 如果你需要在 install 或者 activate 声明周期中打印调试 需要在 console 中开启 preserve log，不然你会只看不到信息，因为每次刷新会重新创建一次 service worker
  ![avatar](/img/prev.webp)

### github 调试

- 由于 service worker 只能在 https 协议中使用，自己的服务器测试不了，刚好可以借用 github 来试验
- github 新建自己的仓库将代码，push 到 github 上，中间又出了点小插曲。

![avatar](/img/token.webp)

- github 在 2021 的中国情人节那天来了个以后登录只能使用 token 登录，这时你就需要去自己 github 上点击自己头像，出现下拉点击 setting，去到设置页面下拉，
  点击 Developer settings，去到开发者设置，点击 Personal access tokens，新建你的 token。
- 要使用 token 从命令行访问仓库，请勾选 repo。
- 要使用 token 从命令行删除仓库，请勾选 delete_repo。
- 之后在你的本地仓库执行：

```js
git remote set-url origin https://<your_token>@github.com/<USERNAME>/<REPO>.git。
例如:git remote set-url origin https://xxx@github.com/xxx/xxx.github.io.git/
```

- 终于可以 github 测试了，可以看到你在断网情况下刷新页面，页面依旧可以运行。
  ![avatar](/img/gitservice.webp)
