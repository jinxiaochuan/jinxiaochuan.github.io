---
title: 浏览器缓存及内容协商
date: 2022-02-12 13:40:50
tags: HTTP
---

#### 浏览器的缓存机制

浏览器的缓存机制也就是我们说的 HTTP 缓存机制，其机制是根据 HTTP 报文的缓存标识进行的。

> 浏览器缓存过程： `强缓存`、`协商缓存`。
> 浏览器缓存位置一般分为四类： `Service Worker`、`Memory Cache`、`Disk Cache`、`Push Cache`

##### 强缓存

强缓存是当我们访问 URL 的时候，不会向服务器发送请求，直接从缓存中读取资源，但是会返回 200 的状态码。

###### 如何设置强缓存？

我们第一次进入页面，请求服务器，然后服务器进行应答，浏览器会根据 response Header 来判断是否对资源进行缓存，如果响应头中 expires、pragma 或者 cache-control 字段，代表这是强缓存，浏览器就会把资源缓存在 memory cache 或 disk cache 中。

第二次请求时，浏览器判断请求参数，如果符合强缓存条件就直接返回状态码 200，从本地缓存中拿数据。否则把响应参数存在 request header 请求头中，看是否符合协商缓存，符合则返回状态码 304，不符合则服务器会返回全新资源。

![browser-cache](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca00bff3081e4cfd993a8f252f4fa23a~tplv-k3u1fbpfcp-watermark.awebp)

> ###### Expires
>
> 是 HTTP1.0 控制网页缓存的字段，值为一个时间戳，准确来讲是格林尼治时间，服务器返回该请求结果缓存的到期时间，意思是，再次发送请求时，如果未超过过期时间，直接使用该缓存，如果过期了则重新请求。
> `缺点：就是它判断是否过期是用本地时间来判断的，本地时间是可以自己修改的。`
>
> ###### Cache-Control
>
> 是 HTTP1.1 中控制网页缓存的字段，当 Cache-Control 都存在时，Cache-Control 优先级更高
>
> - `public`：资源客户端和服务器都可以缓存。
> - `private`：资源只有客户端可以缓存。
> - `no-cache`：客户端缓存资源，但是是否缓存需要经过协商缓存来验证。
> - `no-store`：不使用缓存。
> - `max-age`：最大过期时间（距离当前请求的时间差）
>
> > ![Cache-Control](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f169e913e244d52a44ff1e4185cb9ce~tplv-k3u1fbpfcp-watermark.awebp)
> > Cache-Control 使用了 max-age 相对时间，解决了 expires 的问题
>
> ###### Pragma
>
> 这个是 HTTP1.0 中禁用网页缓存的字段，其取值为 no-cache，和 Cache-Control 的 no-cache 效果一样
>
> ###### Vary(内容协商)
>
> 要了解 Vary 的作用，先得了解 HTTP 的内容协商机制。有时候，同一个 URL 可以提供多份不同的文档，这就要求服务端和客户端之间有一个选择最合适版本的机制，这就是内容协商.
> 服务端根据客户端发送的请求头中某些字段自动发送最合适的版本。可以用于这个机制的请求头字段又分两种：内容协商专用字段（Accept 字段）、其他字段.
> | 请求头字段 | 说明 | 响应头字段
> | :----: | :----- | :----: |
> | Accept | 告知服务器发送何种媒体类型 | Content-Type
> | Accept-Language | 告知服务器发送何种语言 | Content-Language | Content
> |Accept-Charset | 告知服务器发送何种字符集 | Content-Type |
> | Accept-Encoding | 告知服务器采用何种压缩方式 |Content-Encoding |
>
> ```bash
> // 例如客户端发送以下请求头
> // 表示它可以接受任何 MIME 类型的资源；支持采用 gzip、deflate 或 sdch 压缩过的资源；可以接受 zh-CN、en-US 和 en 三种语言，并且 zh-CN 的权重最高（q 取值 0 - 1，最高为 1，最低为 0，默认为 1），服务端应该优先返回语言等于 zh-CN 的版本。
> Accept:*/*
> Accept-Encoding:gzip,deflate,sdch
> Accept-Language:zh-CN,en-US;q=0.8,en;q=0.6
>
> // 浏览器的响应头可能是这样的
> // 表示这个文档确切的 MIME 类型是 text/javascript；文档内容进行了 gzip 压缩；响应头没有 Content-Language 字段，通常说明返回版本的语言正好是请求头 Accept-Language 中权重最高的那个
> Content-Type: text/javascript
> Content-Encoding: gzip
> ```
>
> 上面四个 Accept 字段并不够用，例如要针对特定浏览器如 IE6 输出不一样的内容，就需要用到请求头中的 User-Agent 字段。类似的，请求头中的 Cookie 也可能被服务端用做输出差异化内容的依据。
> 由于客户端和服务端之间可能存在一个或多个中间实体（如缓存服务器），而缓存服务最基本的要求是给用户返回正确的文档。如果服务端根据不同 User-Agent 返回不同内容，而缓存服务器把 IE6 用户的响应缓存下来，并返回给使用其他浏览器的用户，肯定会出问题
> 所以 HTTP 协议规定，如果服务端提供的内容取决于 User-Agent 这样「常规 Accept 协商字段之外」的请求头字段，那么响应头中必须包含 Vary 字段，且 Vary 的内容必须包含 User-Agent。同理，如果服务端同时使用请求头中 User-Agent 和 Cookie 这两个字段来生成内容，那么响应中的 Vary 字段看上去应该是这样的：
>
> ```bash
> Vary: User-Agent, Cookie
> ```
>
> 也就是说 Vary 字段用于列出一个响应字段列表，告诉缓存服务器遇到同一个 URL 对应着不同版本文档的情况时，如何缓存和筛选合适的版本。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c82d0049c3f4f57bf66d8effcb25ed5~tplv-k3u1fbpfcp-watermark.awebp)

##### 缓存位置

强缓存会把资源房放到 memory cache 和 disk cache 中，那什么资源放在 memory cache，什么资源放在 disk cache 中？

![memory cache vs disk cache](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c6020dcbb114111a8e0a09f52d39ab7~tplv-k3u1fbpfcp-watermark.awebp)

查找浏览器缓存时会按顺序查找: `Service Worker` -> `Memory Cache` -> `Disk Cache` -> `Push Cache`。

> ###### Service Worker
>
> 是运行在浏览器背后的独立线程，一般可以用来实现缓存功能。使用 Service Worker 的话，传输协议必须为 HTTPS。因为 Service Worker 中涉及到请求拦截，所以必须使用 HTTPS 协议来保障安全。Service Worker 的缓存与浏览器其他内建的缓存机制不同，它可以让我们自由控制缓存哪些文件、如何匹配缓存、如何读取缓存，并且缓存是持续性的。
>
> ###### Memory Cache
>
> 内存中的缓存，主要包含的是当前中页面中已经抓取到的资源，例如页面上已经下载的样式、脚本、图片等。读取内存中的数据肯定比磁盘快，内存缓存虽然读取高效，可是缓存持续性很短，会随着进程的释放而释放。一旦我们关闭 Tab 页面，内存中的缓存也就被释放了。
>
> ###### Disk Cache
>
> 存储在硬盘中的缓存，读取速度慢点，但是什么都能存储到磁盘中，比之 Memory Cache 胜在容量和存储时效性上。
> 在所有浏览器缓存中，Disk Cache 覆盖面基本是最大的。它会根据 HTTP Herder 中的字段判断哪些资源需要缓存，哪些资源可以不请求直接使用，哪些资源已经过期需要重新请求。并且即使在跨站点的情况下，相同地址的资源一旦被硬盘缓存下来，就不会再次去请求数据。绝大部分的缓存都来自 Disk Cache。
> memory cache 要比 disk cache 快的多。举个例子：从远程 web 服务器直接提取访问文件可能需要 500 毫秒(半秒)，那么磁盘访问可能需要 10-20 毫秒，而内存访问只需要 100 纳秒，更高级的还有 L1 缓存访问(最快和最小的 CPU 缓存)只需要 0.5 纳秒。
> ![prefetch cache](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6216b57ad4cb480884c2b69d0f0ffe26~tplv-k3u1fbpfcp-watermark.awebp)
> 很神奇的，我们又看到了一个 prefetch cache，这个又是什么呢?
>
> > `prefetch cache(预取缓存)`
> > link 标签上带了 prefetch，再次加载会出现。
> > prefetch 是预加载的一种方式，被标记为 prefetch 的资源，将会被浏览器在空闲时间加载。 4. Push Cache
>
> ###### Push Cache(推送缓存)
>
> 是 HTTP/2 中的内容，当以上三种缓存都没有命中时，它才会被使用。它只在会话（Session）中存在，一旦会话结束就被释放，并且缓存时间也很短暂，在 Chrome 浏览器中只有 5 分钟左右，同时它也并非严格执行 HTTP 头中的缓存指令。

##### 协商缓存

协商缓存就是强缓存失效后，浏览器携带缓存标识向服务器发送请求，由服务器根据缓存标识来决定是否使用缓存的过程。

###### 协商缓存生效，返回 304

![协商缓存生效](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f26ab979fcd4df6906a2e9d5e28f56a~tplv-k3u1fbpfcp-watermark.awebp)

###### 协商缓存失效，返回 200 和请求结果

![协商缓存失效](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/449a56554c1e4f0c949e139081a9db4c~tplv-k3u1fbpfcp-watermark.awebp)

> ###### Last-Modified / If-Modified-Since
>
> Last-Modified 是服务器响应请求时，返回该资源文件在服务器最后被修改的时间。
> If-Modified-Since 则是客户端再次发起该请求时，携带上次请求返回的 Last-Modified 值，通过此字段值告诉服务器该资源上次请求返回的最后被修改时间。服务器收到该请求，发现请求头含有 If-Modified-Since 字段，则会根据 If-Modified-Since 的字段值与该资源在服务器的最后被修改时间做对比，若服务器的资源最后被修改时间大于 If-Modified-Since 的字段值，则重新返回资源，状态码为 200；否则则返回 304，代表资源无更新，可继续使用缓存文件。
>
> ###### Etag / If-None-Match
>
> Etag 是服务器响应请求时，返回当前资源文件的一个唯一标识(由服务器生成)。
> If-None-Match 是客户端再次发起该请求时，携带上次请求返回的唯一标识 Etag 值，通过此字段值告诉服务器该资源上次请求返回的唯一标识值。服务器收到该请求后，发现该请求头中含有 If-None-Match，则会根据 If-None-Match 的字段值与该资源在服务器的 Etag 值做对比，一致则返回 304，代表资源无更新，继续使用缓存文件；不一致则重新返回资源文件，状态码为 200。

#### CORS 与 Vary: Origin

在讨论 CORS 与 Vary 关系时，先抛出一个问题：

> 如何避免 CDN 为 PC 端缓存移动端页面？

假设有两个域名访问 `static.shanyue.tech` 的跨域资源

1. `foo.shanyue.tech`，响应头中返回 `Access-Control-Allow-Origin: foo.shanyue.tech`
2. `bar.shanyue.tech`，响应头中返回 `Access-Control-Allow-Origin: bar.shanyue.tech`

看起来一切正常，但平静的水面下波涛暗涌:

「如果 `static.shanyue.tech` 资源被 CDN 缓存，`bar.shanyue.tech` 再次访问资源时，因缓存问题，因此此时返回的是 `Access-Control-Allow-Origin: foo.shanyue.tech`，此时会有跨域问题」

此时，Vary: Origin 就上场了，代表为不同的 Origin 缓存不同的资源，这在各个服务器端 CORS 中间件也能体现出来。

[Koa 关于 CORS 的处理函数](https://github.com/koajs/cors/blob/master/index.js#L54)
```bash
return async function cors(ctx, next) {
  // If the Origin header is not present terminate this set of steps.
  // The request is outside the scope of this specification.
  const requestOrigin = ctx.get('Origin');

  // Always set Vary header
  // https://github.com/rs/cors/issues/10
  ctx.vary('Origin');
}
```

> 服务器端通过响应头 `Origin` 来判断是否为跨域请求，并以此设置多域名跨域，但要加上 `Vary: Origin`。

#### 参考

[彻底理解浏览器的缓存机制](https://mp.weixin.qq.com/s/d2zeGhUptGUGJpB5xHQbOA)
[前端浏览器缓存知识梳理](https://juejin.cn/post/6947936223126093861)
[(1.6w 字)浏览器灵魂之问，请问你能接得住几个？](https://juejin.cn/post/6844904021308735502)
[HTTP 协议中 Vary 的一些研究](https://juejin.cn/post/7000231382731456520)
[浏览器中的跨域问题与 CORS](https://cloud.tencent.com/developer/article/1693677?from=15425)
