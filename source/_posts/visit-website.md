---
title: 从浏览器地址栏输入url之后经历了什么？
date: 2022-03-20 16:29:49
tags: Interview
---

#### 资源是否命中强缓存

1. 如果资源未缓存，发起新请求
2. 如果已缓存，检验是否足够新鲜，足够新鲜直接提供给客户端，否则与服务器进行验证
3. 检验新鲜通常有两个 `HTTP` 头进行控制 `Expires` 和 `Cache-Control`：
   - `HTTP1.0` 提供 `Expires`，值为一个绝对时间表示缓存新鲜日期
   - `HTTP1.1` 增加了 `Cache-Control: max-age=xxx` ，值为以秒为单位的最大新鲜时间

#### DNS 解析

1. 浏览器缓存
2. 本机缓存
3. hosts 文件
4. 路由器缓存
5. ISP DNS 缓存
6. DNS 递归查询（可能存在负载均衡导致每次 IP 不一样）

> DNS 服务器有 3 种类型：`根DNS服务器`、`顶级域（Top-Level Domain, TLD）DNS 服务器` 和 `权威DNS服务器`。它们的层次结构如下图所示：
> ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/408987c0882245cfb1c6a8d853c9d501~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)
>
> - _根 DNS 服务器_
>   首先我们要明确根域名是什么，比如 `www.baidu.com`，有些同学可能会误以为 `com` 就是根域名，其实 `com` 是顶级域名，`www.baidu.com` 的完整写法是 `www.baidu.com.`，最后的这个 `.` 就是根域名。
>   `根DNS服务器`的作用是什么呢？就是管理它的下一级，也就是`顶级域DNS服务器`。通过询问`根DNS服务器`，我们可以知道一个主机名对应的`顶级域DNS服务器`的 `IP` 是多少，从而继续向`顶级域DNS服务器`发起查询请求。
> - _顶级域 DNS 服务器_
>   除了前面提到的 `com` 是顶级域名，常见的顶级域名还有 `cn`、`org`、`edu` 等。`顶级域DNS服务器`，也就是 `TLD`，提供了它的下一级，也就是`权威DNS服务器`的 `IP` 地址。
> - _权威 DNS 服务器_ > `权威DNS服务器`可以返回 `主机 - IP` 的最终映射。
>   ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6bb796a3045e409aabb0f89ad40d3fad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

#### TCP 建联 - 三次握手

1. 客户端发送一个 TCP 的 SYN=1，Seq=X 的包到服务器端口
2. 服务器发回 SYN=1， ACK=X+1， Seq=Y 的响应包
3. 客户端发送 ACK=Y+1， Seq=Z

> _为什么会采用三次握手，若采用二次握手可以吗？ 四次呢？_
> 采用三次握手是为了防止失效的连接请求报文段突然又传送到主机 B，因而产生错误。失效的连接请求报文段是指：主机 A 发出的连接请求没有收到主机 B 的确认，于是经过一段时间后，主机 A 又重新向主机 B 发送连接请求，且建立成功，顺序完成数据传输。考虑这样一种特殊情况，主机 A 第一次发送的连接请求并没有丢失，而是因为网络节点导致延迟达到主机 B，主机 B 以为是主机 A 又发起的新连接，于是主机 B 同意连接，并向主机 A 发回确认，但是此时主机 A 根本不会理会，主机 B 就一直在等待主机 A 发送数据，导致主机 B 的资源浪费。
>
> _采用两次握手为什么不行？_
> 如客户端发出连接请求，但因连接请求报文丢失而未收到确认，于是客户端再重传一次连接请求。后来收到了确认，建立了连接。数据传输完毕后，就释放了连接，客户端共发出了两个连接请求报文段，其中第一个丢失，第二个到达了服务端，但是第一个丢失的报文段只是在某些网络结点长时间滞留了，延误到连接释放以后的某个时间才到达服务端，此时服务端误认为客户端又发出一次新的连接请求，于是就向客户端发出确认报文段，同意建立连接，不采用三次握手，只要服务端发出确认，就建立新的连接了，此时客户端忽略服务端发来的确认，也不发送数据，则服务端一致等待客户端发送数据，浪费资源。
>
> ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/11/9/d8bf92c7906718271fdb8b0d2d5fe5b4~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)
>
> ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/6/26/1643a1dd6df4813b~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 服务端处理请求并响应

1. 服务器接受请求并解析，将请求转发到服务程序，如虚拟主机使用 HTTP Host 头部判断请求的服务程序
2. 服务器检查 HTTP 请求头是否包含缓存验证信息，如果验证缓存新鲜，返回 304 等对应状态码
3. 处理程序读取完整请求并准备 HTTP 响应，可能需要查询数据库等操作
4. 服务器将响应报文通过 TCP 连接发送回浏览器

#### TCP 断联 - 四次挥手

1. 主动方发送 Fin=1， Ack=Z， Seq= X 报文
2. 被动方发送 ACK=X+1， Seq=Z 报文
3. 被动方发送 Fin=1， ACK=X， Seq=Y 报文
4. 主动方发送 ACK=Y， Seq=X 报文

> _挥手为什么需要四次？_
> 因为当服务端收到客户端的 SYN 连接请求报文后，可以直接发送 SYN+ACK 报文。其中 ACK 报文是用来应答的，SYN 报文是用来同步的。但是关闭连接时，当服务端收到 FIN 报文时，很可能并不会立即关闭 SOCKET，所以只能先回复一个 ACK 报文，告诉客户端，"你发的 FIN 报文我收到了"。只有等到我服务端所有的报文都发送完了，我才能发送 FIN 报文，因此不能一起发送。故需要四次挥手。
>
> _四次挥手释放连接时，等待 2MSL 的意义?_
>
> > MSL 是 Maximum Segment Lifetime 的英文缩写，可译为“最长报文段寿命”，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。

> 为了保证客户端发送的最后一个 ACK 报文段能够到达服务器。因为这个 ACK 有可能丢失，从而导致处在 LAST-ACK 状态的服务器收不到对 FIN-ACK 的确认报文。服务器会超时重传这个 FIN-ACK，接着客户端再重传一次确认，重新启动时间等待计时器。最后客户端和服务器都能正常的关闭。假设客户端不等待 2MSL，而是在发送完 ACK 之后直接释放关闭，一但这个 ACK 丢失的话，服务器就无法正常的进入关闭连接状态。
>
> ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/8/16da9fd28b49f652~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp) > ![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/6/26/1643a20296de1ff0~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

#### 浏览器处理响应

1. 浏览器检查响应状态吗：是否为 1XX，3XX， 4XX， 5XX，这些情况处理与 2XX 不同
2. 如果资源可缓存，进行缓存
3. 对响应进行解码（例如 gzip 压缩）
4. 根据资源类型决定如何处理（假设资源为 HTML 文档）

#### 浏览器渲染

##### 构建 DOM 树

- Tokenizing：根据 HTML 规范将字符流解析为标记
- Lexing：词法分析将标记转换为对象并定义属性和规则
- DOM construction：根据 HTML 标记关系将对象组成 DOM 树

##### 构建 CSSOM 树

- Tokenizing：字符流转换为标记流
- Node：根据标记创建节点
- CSSOM：节点创建 CSSOM 树

##### [根据 DOM 树和 CSSOM 树构建渲染树](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/render-tree-construction)

- 从 DOM 树的根节点遍历所有可见节点，不可见节点包括：
  - 1）script,meta 这样本身不可见的标签。
  - 2)被 css 隐藏的节点，如 display: none
- 对每一个可见节点，找到恰当的 CSSOM 规则并应用
- 发布可视节点的内容和计算样式

##### js 解析

- 浏览器创建 `Document` 对象并解析 `HTML`，将解析到的元素和文本节点添加到文档中，此时 `document.readystate` 为 `loading`
- `HTML` 解析器遇到没有 `async` 和 `defer` 的 `script` 时，将他们添加到文档中，然后执行行内或外部脚本。这些脚本会同步执行，并且在脚本下载和执行时解析器会暂停。这样就可以用 `document.write()` 把文本插入到输入流中。同步脚本经常简单定义函数和注册事件处理程序，他们可以遍历和操作 `script` 和他们之前的文档内容
- 当解析器遇到设置了 `async` 属性的 `script` 时，开始下载脚本并继续解析文档。脚本会在它下载完成后尽快执行，但是解析器不会停下来等它下载。异步脚本禁止使用 `document.write()`，它们可以访问自己 `script` 和之前的文档元素
- 当文档完成解析，`document.readState` 变成 `interactive`
- 所有 `defer` 脚本会按照在文档出现的顺序执行，延迟脚本能访问完整文档树，禁止使用 `document.write()`
- 浏览器在 `Document` 对象上触发 `DOMContentLoaded` 事件
- 此时文档完全解析完成，浏览器可能还在等待如图片等内容加载，等这些内容完成载入并且所有异步脚本完成载入和执行，`document.readState` 变为 `complete`,`window` 触发 `load` 事件

#### 参考

[字节面试被虐后，是时候搞懂 DNS 了](https://juejin.cn/post/6990344840181940261)
[从浏览器地址栏输入 url 到显示页面的步骤以 http 为例](https://github.com/jinxiaochuan/FE-interview#%E4%BB%8E%E6%B5%8F%E8%A7%88%E5%99%A8%E5%9C%B0%E5%9D%80%E6%A0%8F%E8%BE%93%E5%85%A5url%E5%88%B0%E6%98%BE%E7%A4%BA%E9%A1%B5%E9%9D%A2%E7%9A%84%E6%AD%A5%E9%AA%A4%E4%BB%A5http%E4%B8%BA%E4%BE%8B)
[从输入 URL 到浏览器显示页面过程中都发生了什么？](https://juejin.cn/post/6905931622374342670)
[跟着动画来学习 TCP 三次握手和四次挥手](https://juejin.cn/post/6844903625513238541)
[面试官，不要再问我三次握手和四次挥手](https://juejin.cn/post/6844903958624878606)
