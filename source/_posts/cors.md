---
title: CORS vs HSTS
date: 2022-02-12 16:54:51
tags: HTTP
---

#### 浏览器同源策略

所谓同源就是浏览器的一个安全机制,不同源的客户端脚本没有在明确授权的情况下,不能读写对方资源。由于存在同源策略的限制,而又有需要跨域的业务,所以就有了 CORS 的出现。

当资源位于不同协议、子域或端口的站点时，这个请求就是跨域的

![同源策略](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d31d71ca8cd042009096df091777b014~tplv-k3u1fbpfcp-watermark.awebp)

#### CORS 与 HSTS

HSTS 全称：HTTP Strict Transport Security，意译：HTTP 严格传输安全，是一个 Web 安全策略机制。

> ##### HSTS 解决什么问题？
>
> 它解决的是：网站从 Http 转跳到 Https 时，可能出现的安全问题。
> Client 从 Http 切换到 Https 前是明文传输，因此是可以被 Man-In-The-Middle 劫持的，如下流程：
>
> ![Http->Https](https://www.freesion.com/images/100/bf3056e8c6b99883df8f0557a1551584.JPEG)
>
> ##### HSTS 如何解决?
>
> 要解决从 Http 切换到 Https 被劫持的问题，只要一开始就没有 Http 请求即可，流程如下：
>
> ![HSTS](https://www.freesion.com/images/675/70738984e0e69d40c8b43d6c4a164843.JPEG)
>
> ##### HSTS 如何知道哪些请求该转为 HTTPS，哪些不该转?
>
> > `HSTS HEADER(Strict-Transport-Security)`
> >
> > 最近一次请求的 HTTPS 响应中（RESPONSE）中，带上 HSTS HEADER：
> >
> > ```bash
> > Strict-Transport-Security: <max-age=>[; includeSubDomains][; preload]
> > ```
> >
> > `HSTS PRELOAD LIST`
> >
> > HSTS HEADER(Strict-Transport-Security) 还是有个漏洞：
> >
> > - 如果第一次访问网站 A 就被劫持了，哪方案 1 岂不白搭？
> > - 清 Cookies 或者 HSTS Header 过期了，下次访问岂不又风险重重？
> >   基于以上问题，就需要方案 2（HSTS Preload List）：
> >
> > 官方说明：
> >
> > ```bash
> > This is a list of sites that are hardcoded into Chrome as being HTTPS only.
> > HSTS Preload List 是一个站点列表，它被 hardcode 写入 Chrome 中，列表中的站点将会默认使用 HTTPS 进行访问。
> >
> > Most major browsers (Chrome, Firefox, Opera, Safari, IE 11 and Edge) also have HSTS preload lists based on the Chrome list. (See the HSTS compatibility matrix.)
> > 主流浏览器（Firefox, Opera, Safari, IE 11 and Edge）都有和 Chrome 一样的 HSTS Preload List。
> > ```
>
> ##### HSTS 可能导致 CORS 产生跨域问题
>
> HSTS (HTTP Strict Transport Security) 为了避免 HTTP 跳转到 HTTPS 时遭受潜在的中间人攻击，由浏览器本身控制到 HTTPS 的跳转。如同 CORS 一样，它也是有一个服务器的响应头 `Strict-Transport-Security: max-age=5184000` 来控制，此时浏览器访问该域名时，会使用 307 Internal Redirect，无需服务器干涉，自动跳转到 HTTPS 请求。
>
> 「如果前端访问 HTTP 跨域请求，此时浏览器通过 HSTS 跳转到 HTTPS，但浏览器不会给出相应的 CORS 响应头部，就会发生跨域问题。」
>
> ```bash
> GET / HTTP/1.1
> Host: shanyue.tech
> Origin: http://shanyue.tech
> User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
> Access to XMLHttpRequest at 'xxx' from origin 'xxx' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
> ```

#### CORS

CORS 即跨域资源共享 (Cross-Origin Resource Sharing, CORS)。简而言之，就是在服务器端的响应中加入几个标头，使得浏览器能够跨域访问资源。

这个响应头的字段设置就是：`Access-Control-Allow-Origin: *`

浏览器将 CORS 请求分成两类：`简单请求`和`非简单请求`

> ###### 简单请求
>
> Method: 请求的方法是 GET、POST 及 HEAD
> Header: 请求头是 Content-Type (有限制)、Accept-Language、Content-Language 等
> Content-Type: 请求类型是 application/x-www-form-urlencoded、multipart/form-data 或 text/plain
>
> ###### 非简单请求
>
> 除简单请求外，均为非简单请求。一般需要开发者主动构造，在项目中常见的 Content-Type: application/json 及 Authorization: <token> 为典型的「非简单请求」
>
> ###### 预检请求(preflight request)
>
> 非简单请求的 CORS 请求是会在正式通信之前进行一次预检请求，"预检"使用的请求方法是 OPTIONS , 表示这个请求是用来询问的
>
> ```bash
> // 跨域请求
> var url = 'http://localhost:2333/cors';
> var xhr = new XMLHttpRequest();
> xhr.open('PUT', url, true);
> xhr.setRequestHeader('X-Custom-Header', 'value');
> xhr.send();
> ```
>
> 由于上面的代码使用的是 PUT 方法,并且发送了一个自定义头信息.所以是一个非简单请求,当浏览器发现这是一个非简单请求的时候,会自动发出预检请求,看看服务器可不可以接收这种请求,下面是"预检"的 HTTP 头信息
>
> ```bash
> // 跨域请求 预检 request headers
> OPTIONS /cors HTTP/1.1
> Origin: localhost:2333
> Access-Control-Request-Method: PUT // 表示使用的什么HTTP请求方法
> Access-Control-Request-Headers: X-Custom-Header // 表示浏览器发送的自定义字段
> Host: localhost:2332
> Accept-Language: zh-CN,zh;q=0.9
> Connection: keep-alive
> User-Agent: Mozilla/5.0...
> ```
>
> 预检请求后的回应，服务器收到"预检"请求以后，检查了 Origin、Access-Control-Request-Method 和 Access-Control-Request-Headers 字段以后，确认允许跨源请求，就可以做出回应。
>
> ```bash
> // 跨域请求 预检 response headers
> HTTP/1.1 200 OK
> Date: Mon, 01 Dec 2008 01:15:39 GMT
> Server: Apache/2.0.61 (Unix)
> Access-Control-Allow-Origin: http://localhost:2332 // 表示http://localhost:2332可以访问数据
> Access-Control-Allow-Methods: GET, POST, PUT
> Access-Control-Allow-Headers: X-Custom-Header
> Content-Type: text/html; charset=utf-8
> Content-Encoding: gzip
> Content-Length: 0
> Keep-Alive: timeout=2, max=100
> Connection: Keep-Alive
> Content-Type: text/plain
> ```
>
> ###### CORS Response Headers
>
> `Access-Control-Allow-Origin`: 可以把资源共享给那些域名，支持 \* 及 特定域名
> `Access-Control-Allow-Credentials`: 请求是否可以带 cookie
> `Access-Control-Allow-Methods`: 请求所允许的方法, 「用于预检请求中」
> `Access-Control-Allow-Headers`: 请求所允许的头，「用于预检请求中」
> `Access-Control-Expose-Headers`: 那些头可以在响应中列出
> `Access-Control-Max-Age`: 预检请求的缓存时间
>
> ###### [Koa CORS 中间件原理](https://github.com/koajs/cors/blob/master/index.js)
>
> > 必须校验是否有 `Origin` 请求头，跨域请求一定会有 `Origin` 请求头，`@koa/cors` 中间件源码 `origin = options.origin || requestOrigin` 可知，优先取 `options.origin`，默认以当前请求的 `Origin` 请求头作为 `Access-Control-Allow-Origin` 的响应头信息，保证当前请求允许跨域。
>
> ```bash
> module.exports = function(options) {
>  const defaults = {
>    allowMethods: 'GET,HEAD,PUT,POST,DELETE,PATCH',
>  };
>
>  options = {
>    ...defaults,
>    ...options,
>  };
>
>  // ...
>
>  return async function cors(ctx, next) {
>    // If the Origin header is not present terminate this set of steps.
>    // The request is outside the scope of this specification.
>    const requestOrigin = ctx.get('Origin');
>
>    // Always set Vary header
>    // https://github.com/rs/cors/issues/10
>    ctx.vary('Origin');
>
>    if (!requestOrigin) return await next();
>
>    let origin;
>    if (typeof options.origin === 'function') {
>      origin = options.origin(ctx);
>      if (origin instanceof Promise) origin = await origin;
>      if (!origin) return await next();
>    } else {
>      origin = options.origin || requestOrigin;
>    }
>
>    let credentials;
>    if (typeof options.credentials === 'function') {
>      credentials = options.credentials(ctx);
>      if (credentials instanceof Promise) credentials = await credentials;
>    } else {
>      credentials = !!options.credentials;
>    }
>
>    const headersSet = {};
>
>    function set(key, value) {
>      ctx.set(key, value);
>      headersSet[key] = value;
>    }
>
>    if (ctx.method !== 'OPTIONS') {
>      // Simple Cross-Origin Request, Actual Request, and Redirects
>      set('Access-Control-Allow-Origin', origin);
>
>      if (credentials === true) {
>        set('Access-Control-Allow-Credentials', 'true');
>      }
>
>      if (options.exposeHeaders) {
>        set('Access-Control-Expose-Headers', options.exposeHeaders);
>     }
>
>      if (!options.keepHeadersOnError) {
>        return await next();
>      }
>      try {
>        return await next();
>      } catch (err) {
>        const errHeadersSet = err.headers || {};
>        const varyWithOrigin = vary.append(errHeadersSet.vary || errHeadersSet.Vary || '', 'Origin');
>        delete errHeadersSet.Vary;
>
>        err.headers = {
>          ...errHeadersSet,
>          ...headersSet,
>          ...{ vary: varyWithOrigin },
>        };
>        throw err;
>      }
>    } else {
>      // Preflight Request
>
>      // If there is no Access-Control-Request-Method header or if parsing failed,
>      // do not set any additional headers and terminate this set of steps.
>      // The request is outside the scope of this specification.
>      if (!ctx.get('Access-Control-Request-Method')) {
>        // this not preflight request, ignore it
>        return await next();
>      }
>
>      ctx.set('Access-Control-Allow-Origin', origin);
>
>      if (credentials === true) {
>        ctx.set('Access-Control-Allow-Credentials', 'true');
>      }
>
>      if (options.maxAge) {
>        ctx.set('Access-Control-Max-Age', options.maxAge);
>      }
>
>      if (options.allowMethods) {
>        ctx.set('Access-Control-Allow-Methods', options.allowMethods);
>      }
>
>      let allowHeaders = options.allowHeaders;
>      if (!allowHeaders) {
>        allowHeaders = ctx.get('Access-Control-Request-Headers');
>      }
>      if (allowHeaders) {
>        ctx.set('Access-Control-Allow-Headers', allowHeaders);
>      }
>
>      ctx.status = 204;
>    }
>  };
> };
> ```

#### CORS 与 Vary: Origin

当请求网络静态资源为跨域资源（常见的为 cdn 静态资源，如 js、css、images 等）时，通常是会设置相关协商缓存响应头（`Last-Modified`、`Etag`）。
这种一旦相同的资源链接需要根据不同的请求头（如`User-Agent`）响应不同的静态资源（PC、Mobile）时，可能会导致缓存资源响应错乱，此时必须设置 `Vary: Origin, User-Agent`，即代表为不同的 `Origin` 或 `User-Agent` 缓存不同的资源。

详见 [浏览器缓存及内容协商](https://jinxiaochuan.github.io/post/broswer-cache)

#### 参考

[浏览器中的跨域问题与 CORS](https://cloud.tencent.com/developer/article/1693677?from=15425)
[面试官问我 CORS 跨域，我直接一套操作斩杀！](https://juejin.cn/post/6983852288091619342)
[浅析 HSTS](https://www.freesion.com/article/61111103085/)
[简单易懂 HSTS，你需要它！](https://juejin.cn/post/6844903952211771405)
[【安全】HSTS - 强制客户端（如浏览器）使用 HTTPS 与服务器创建连接](https://juejin.cn/post/6844904086966370318)
