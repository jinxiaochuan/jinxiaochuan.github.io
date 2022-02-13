---
title: Chrome 80+
date: 2022-02-13 14:20:14
tags:
---

#### Chrome 80 策略更新

Chrome 80 稳定版（版本号 v80.0.3987.87）已正式面向 Windows、macOS、Linux、Android 和 iOS 全平台推送

##### 混合内容强制 HTTPS

混合内容是指 https 页面下有非 https 资源时，浏览器的加载策略。

在 Chrome 80 中，如果你的页面开启了 https，同时你在页面中请求了 http 的音频和视频资源，这些资源将将自动升级为 https ，并且默认情况下，如果它们无法通过 https 加载，Chrome 将阻止它们。这样就会造成一些未支持 https 协议的资源加载失败。

如果你想临时访问这些资源，你可以通过更改下面的浏览器设置来访问：

> 1. 单击地址栏上的锁定图标并选择 “站点设置”
> 2. 将 "隐私设置和安全性" 中的 "不安全内容" 选择为 "允许"
> 3. 你还可以通过设置 `StricterMixedContentTreatmentEnabled` 策略来控制这些变化：
>
>    > 此策略控制浏览器中混合内容（HTTPS 站点中的 HTTP 内容）的处理方式。如果该政策设置为 true 或未设置，则音频和视频混合内容将自动升级为 HTTPS（即，URL 将被重写为 HTTPS，如果资源不能通过 HTTPS 获得，则不会进行回退），并且将显示“不安全”警告在网址列中显示图片混合内容。如果该策略设置为 false，则将禁用音频和视频的自动升级，并且不会显示图像警告。该策略不影响音频，视频和图像以外的其他类型的混合内容。
>
>    但是以上策略是一个临时策略，将在 Chrome 84 中删除。更合理的方式是你需要推动全站资源开启 HTTPS。Chrome 也是推荐大家这么做的

##### 强推 SameSite Cookie

SameSite 是 Chrome 51 版本为浏览器的 Cookie 新增的了一个属性， SameSite 阻止浏览器将此 Cookie 与跨站点请求一起发送。其主要目标是降低跨源信息泄漏的风险。同时也在一定程度上阻止了 CSRF（Cross-site request forgery 跨站请求伪造）

Cookie 往往用来存储用户的身份信息，恶意网站可以设法伪造带有正确 Cookie 的 HTTP 请求，这就是 CSRF 攻击。

> SameSite 可以避免跨站请求发送 Cookie，有以下三个属性：
>
> - _Strict_
>   Strict 是最严格的防护，将阻止浏览器在所有跨站点浏览上下文中将 Cookie 发送到目标站点，即使在遵循常规链接时也是如此。因此这种设置可以阻止所有 CSRF 攻击。然而，它的用户友好性太差，即使是普通的 GET 请求它也不允许通过。
>   例如，对于一个普通的站点，这意味着如果一个已经登录的用户跟踪一个发布在公司讨论论坛或电子邮件上的网站链接，这个站点将不会收到 Cookie ，用户访问该站点还需要重新登陆。
>   不过，具有交易业务的网站很可能不希望从外站链接到任何交易页面，因此这种场景最适合使用 strict 标志。
> - _Lax_
>   对于允许用户从外部链接到达本站并使用已有会话的网站站，默认的 Lax 值在安全性和可用性之间提供了合理的平衡。Lax 属性只会在使用危险 HTTP 方法发送跨域 Cookie 的时候进行阻止，例如 POST 方式。
>   例如，一个用户在 A 站点 点击了一个 B 站点（GET 请求），而假如 B 站点 使用了 Samesite-cookies=Lax，那么用户可以正常登录 B 站点。相对地，如果用户在 A 站点提交了一个表单到 B 站点（POST 请求），那么用户的请求将被阻止，因为浏览器不允许使用 POST 方式将 Cookie 从 A 域发送到Ｂ域。
>   ![SameSite Lax](https://pic2.zhimg.com/v2-2c33b6a018f9dd194404c899ffb7c915_r.jpg)
> - _None_
>   浏览器会在同站请求、跨站请求下继续发送 Cookies，不区分大小写。

以上更新可能对以下功能造成影响：

- 跨域名登陆失效
- jsonp 获取数据失效
- iframe 嵌套的页面打不开或异常
- 部分客户端未改造导致各种数据获取异常

##### 正式支持 JavaScript Optional chaining & Nullish coalescing

###### JavaScript Optional chaining

```bash
// 对象式写法
// 通常写法
const nestedProp = obj.first && obj.first.second;
// 支持JavaScript Optional chaining后我们可以不用再做无谓的判断逻辑了
const nestedProp = obj.first?.second;

// 数组式写法
// 通常写法
const firstEl = arr && Array.isArray(arr) && arr[0];
// 支持JavaScript Optional chaining后
const firstEl = arr?.[0];

// 函数式写法
// 通常写法
const result = someInterface.customMethod && someInterface.customMethod();
// 支持JavaScript Optional chaining后
const result = someInterface.customMethod?.();
```

> 注：如果在开发的时候，需要兼容一些旧版本的浏览器的话可以考虑引入 `@babel/plugin-proposal-optional-chaining` 插件

###### Nullish coalescing

```bash
/***************** Example 1 *******************/
const foo;

//  在下面代码执行完之后foo还是undefined，并没有被赋值，bar的结果一定会是'hello'
const bar = foo || 'Hello!';

/***************** Example 2 *******************/

const num = 0;
const txt = '';

const qty = num || 42;
const team = txt || 'AIPE-FE';
console.log(qty);  // 42 and not 0
console.log(team); // "AIPE-FE" and not ''

/***************** Example 3 *******************/

let txt = '';

const team = txt || 'AIPE-FE';
console.log(team); // 'AIPE-FE'

let result = txt ?? 'AIPE-FE';
console.log(result); // '' (这个时候txt就不再是''了，而是被赋值成'AIPE-FE');

/***************** Example 4 *******************/
const nullValue = null;
const emptyText = '';
const someNumber = 42;

const valA = nullValue ?? 'default for A';
const valB = emptyText ?? 'default for B';
const valC = someNumber ?? 0;

console.log(valA); // 'default for A'
console.log(valB); // '' (as the empty string is not null or undefined)
console.log(valC); // 42
```

> 注：如果在开发的时候，需要兼容一些旧版本的浏览器的话可以考虑引入 `@babel/plugin-proposal-nullish-coalescing-operator` 插件

##### Favicon 图标支持 SVG 格式

这个升级点就不用多说了

```bash
new CopyWebpackPlugin([
    {
      // 我们现在可以输出svg格式的favicon了
      from: 'images/favicon-32.svg'
    }
])
```

##### 移除对 FTP 的支持

![Remove FTP](https://pic3.zhimg.com/v2-570fa0e42bdcdc9c6b2c14dead0ed82e_r.jpg)

Google Chrome 当前 FTP 的实现不支持加密连接（FTPS），也没有代理。 FTP 在浏览器中的使用率非常低，以致无法再投资于改进现有的 FTP 客户端。此外，所有受影响的平台上都提供了功能更强大的 FTP 客户端。 Google Chrome 72+删除了对通过 FTP 提取文档子资源和呈现顶级 FTP 资源的支持。当前，导航到 FTP URL 会导致显示目录列表或下载，具体取决于资源的类型。 Google Chrome 74+中的一个错误导致放弃了对通过 HTTP 代理访问 FTP URL 的支持。对 FTP 的代理支持已在 Google Chrome 76 中完全删除。 Google Chrome 的 FTP 实施的其余功能仅限于显示目录列表或通过未加密的连接下载资源。我们想弃用并删除此剩余功能，而不是维护不安全的 FTP 实现。

总的来说：chrome 80+ 以后，google chrome 团队更 focus 的点就是安全相关的问题

##### Web workers 中支持 ES modules

Module Workers 是一种适用于 Web Worker 的新模式-得益于 JavaScript 模块化的优势。 Worker 构造函数现在可以接受一个{type：“ module”}选项，该选项更改了脚本的加载和执行方式，用于匹配

```bash
<script type="module">
    const worker = new Worker('worker.js', {
        type: 'module'
    });
</script>
```

详见 [Threading the web with module workers](https://web.dev/module-workers/)

#### Chrome 86 策略更新

Chrome 86 在 2020 年 10 月推出了稳定版，现已全面应用于 Android、Chrome OS、Linux、macOS 和 Windows 等平台

##### 文件系统访问

通过调用 showOpenFilePicker 方法，你可以唤起文件选择窗口，进而通过返回的文件句柄对文件进行读写。

```bash
async function getFileHandle() {
  const opts = {
    types: [
      {
        description: 'Text Files',
        accept: {
          'text/plain': ['.txt', '.text'],
          'text/html': ['.html', '.htm']
        }
      }
    ]
  };
  return await window.showOpenFilePicker(opts);
}
```

##### 全面阻止所有非 HTTPS 混合内容下载

HTTPS 混合内容错误是指初始网页通过安全的 HTTPS 链接加载，但页面中其他资源，比如图像，视频，样式表，脚本却通过不安全的 HTTP 链接加载，这样就会出现混合内容错误，也就是不安全因素。

攻击者可拦截不安全的下载地址，将程序替换成恶意软件、甚至访问更多的敏感信息。为管控这些风险，谷歌最终还是决定在 Chrome 中禁止加载不安全资源。

从 M86 开始，图片类型的请求，会自动升级到 HTTPS，并且没有 HTTP 的降级，Audio/Video 类型的请求早在 M80 就开始进行了自动升级

##### replaceChildren

目前，要想替换某 DOM 节点下的全部子节点，必须要先通过 innerHTML 或 removeChild 删除全部子节点，然后再逐个添加，比较麻烦。为此，Chrome 支持了 replaceChildren 方法，可以用参数中的子节点列表替换原有的全部子节点，代码如下：

```bash
  parentNode.replaceChildren(newChildren);
```

##### 更醒目的 HTTP 安全警告

在我们访问 HTTPS 网页时，地址栏最左侧会显示一个锁定图标来表明当前网站是安全的，但如果 HTTPS 网页中嵌入的是并不安全的 HTTP 表单，浏览器则不会给出任何提示信息。而实际上已经有钓鱼网站通过这种方式来盗取用户的敏感信息了。

所以在 Chrome 86 中，如果 HTTPS 的网页中嵌入了不安全的 HTTP 表单，表单字段下方会有极为醒目的「此表单不安全」文本提示。

![HTTP Secure Warning](https://pic1.zhimg.com/v2-40d5adbb662be961244eb8c17b03fb64_r.jpg)

##### 后台标签页更省电

如果一个标签页在后台运行了五分钟以上，这个页面就会被暂时冻结，相应的 CPU 使用也会被限制在 1% 左右；如果页面支持自动刷新，唤醒时间被限制在每一分钟一次。



#### 参考

[Chrome 80 版本到底升级了些什么](https://zhuanlan.zhihu.com/p/107126906)
[两个你必须要重视的 Chrome 80 策略更新！！！](https://cloud.tencent.com/developer/article/1590217)
[2020 年 10 月 Chrome 86 重要更新解读](https://zhuanlan.zhihu.com/p/281009581)
[Threading the web with module workers](https://web.dev/module-workers/)
