---
title: C/C++ SDK 使用指南 | 七牛云存储
---

# C/C++ SDK 使用指南

本SDK使用符合 C89 标准的 C 语言实现。由于 C 语言的普适性，原则上此 SDK 可以跨所有主流平台，不仅可以直接在 C 和 C++ 的工程中使用，也可以用于与 C 语言交互性较好的语言中，比如 C#（使用P/Invoke交互）、Java（使用JNI交互）、Lua 等。本开发指南假设开发者使用的开发语言是 C/C++。

SDK 下载地址：<https://github.com/qiniu/c-sdk/tags>

**文档大纲**

- [概述](#overview)
- [准备开发环境](#prepare)
    - [环境依赖](#dependences)
    - [ACCESS_KEY 和 SECRET_KEY](#appkey)
- [初始化环境与清理](#init)
- [C-SDK惯例](#convention)
    - [HTTP客户端](#http-client)
    - [错误处理与调试](#error-handling)
- [上传文件](#io-put)
    - [上传流程](#io-put-flow)
    - [上传策略](#io-put-policy)
    - [断点续上传](#resumable-io-put)
- [下载文件](#io-get)
    - [下载私有文件](#io-get-private)
    - [HTTPS 支持](#io-https-get)
    - [断点续下载](#resumable-io-get)
- [资源操作](#rs)
    - [获取文件信息](#rs-stat)
    - [删除文件](#rs-delete)
    - [复制/移动文件](#rs-copy-move)
    - [批量操作](#rs-batch)

<a name="overview"></a>

## 概述

七牛云存储的 C 语言版本 SDK（本文以下称 C-SDK）是对七牛云存储API协议的一层封装，以提供一套对于 C/C++ 开发者而言简单易用的原生C语言函数。C/C++ 开发者在对接 C-SDK 时无需理解七牛云存储 API 协议的细节，原则上也不需要对 HTTP 协议和原理做非常深入的了解，但如果拥有基础的 HTTP 知识，对于出错场景的处理可以更加高效。

C-SDK 以开源方式提供。开发者可以随时从本文档提供的下载地址查看和下载 SDK 的源代码，并按自己的工程现状进行合理使用，比如编译为静态库或者动态库后进行链接，或者直接将 SDK 的源代码加入到自己的工程中一起编译，以保持工程设置的简单性。

由于 C 语言的通用性，C-SDK 被设计为同时适合服务器端和客户端使用。服务端是指开发者自己的业务服务器，客户端是指开发者提供给终端用户的软件，通常运行在 iPhone/iPad/Android 移动设备，或者运行在 Windows/Mac/Linux 这样的桌面平台上。服务端因为有七牛颁发的 AccessKey/SecretKey，可以做很多客户端做不了的事情，比如删除文件、移动/复制文件等操作。除非开发者将文件设置为公开，客服端操作私有文件需要获得服务端的授权。客户端上传文件需要获得服务端颁发的 [uptoken（上传授权凭证）](http://docs.qiniutek.com/v3/api/io/#upload-token)，客户端下载文件（包括下载处理过的文件，比如下载图片的缩略图）需要获得服务端颁发的 [dntoken（下载授权凭证）](http://docs.qiniutek.com/v3/api/io/#private-download)。

注意从 v5.0.0 版本开始，我们对 SDK 的内容进行了精简。所有管理操作，比如：创建Bucket、删除Bucket、为Bucket绑定域名（publish）、设置数据处理的样式分隔符（fop seperator）、新增数据处理样式（fop style）等都去除了，统一建议到[开发者后台](https://dev.qiniutek.com/)来完成。另外，此前服务端还有自己独有的上传 API，现在也推荐统一成基于客户端上传的工作方式。

从内容上来说，C-SDK 主要包含如下几方面的内容：

* 公共部分，所有用况下都用到：qiniu/base.c, qiniu/conf.c, qiniu/oauth2.c
* 客户端上传文件：qiniu/base_io.c, qiniu/io.c
* 客户端断点续上传：qiniu/base_io.c, qiniu/resumable_io.c
* 数据处理：qiniu/fop.c
* 服务端操作：qiniu/oauth2_digest.c (授权), qiniu/rs.c (资源操作), qiniu/rs_token.c (uptoken/dntoken颁发)


<a name="prepare"></a>

## 准备开发环境

<a name="dependences"></a>

### 环境依赖

C-SDK 使用 [cURL](http://curl.haxx.se/) 进行网络相关操作。无论是作为客户端还是服务端，都需要依赖 [cURL](http://curl.haxx.se/)。如果作为服务端，C-SDK 因为需要用 HMAC 进行数字签名做授权（简称签名授权），所以依赖了 [OpenSSL](http://www.openssl.org/) 库。C-SDK 并没有带上这两个外部库，因此在使用 C-SDK 之前需要先确认您的当前开发环境中是否已经安装了这所需的外部库，并且已经将它们的头文件目录和库文件目录都加入到了项目工程的设置。

在主流的*nix环境下，通常 [cURL](http://curl.haxx.se/) 和 [OpenSSL](http://www.openssl.org/) 都已经随系统默认安装到`/usr/include`和`/usr/lib`目录下。如果你的系统还没有这些库，请自行安装。如何安装这些第三方库不在本文讨论范围，请自行查阅相关文档。

如果你使用 gcc 进行编译，服务端典型的链接选项是：`-lcurl -lssl -lcrypto -lm`，客户端则是：`-lcurl -lm`。

如果在项目构建过程中出现环境相关的编译错误和链接错误，请确认这些选项是否都已经正确配置，以及所依赖的库是否都已经正确的安装。


<a name="appkey"></a>

### ACCESS_KEY 和 SECRET_KEY

如果你的服务端采用 C-SDK，那么使用 C-SDK 前，您需要拥有一对有效的 AccessKey 和 SecretKey 用来进行签名授权。可以通过如下步骤获得：

1. [开通七牛开发者帐号](https://dev.qiniutek.com/signup)
2. [登录七牛开发者自助平台，查看 AccessKey 和 SecretKey](https://dev.qiniutek.com/account/keys) 。

C-SDK 的 conf.h 文件中声明了对应的两个变量：`QINIU_ACCESS_KEY`和`QINIU_SECRET_KEY`。你需要在启动程序之初初始化这两个变量为七牛颁发的 AccessKey 和 SecretKey。


<a name="init"></a>

## 初始化环境与清理

对于服务端而言，常规程序流程是：

    @gist(gist/server.c#init)

对于客户端而言，常规程序流程是：

    @gist(gist/client.c#init)

两者主要的区别在于：

1. 客户端没有 `QINIU_ACCESS_KEY`, `QINIU_SECRET_KEY` 变量（不需要初始化）。
2. 客户端没有签名授权，所以初始化 `Qiniu_Client` 对象应该用 `Qiniu_Client_InitNoAuth` 而不是 `Qiniu_Client_Init`。


<a name="convention"></a>

## C-SDK 惯例

C 语言是一个非常底层的语言，相比其他高级语言来说，它的代码通常看起来会更啰嗦。为了尽量让大家理解我们的 C-SDK，这里需要解释下我们在 SDK 中的一些惯例做法。

<a name="http-client"></a>

### HTTP 客户端

在 C-SDK 中，HTTP 客户端叫`Qiniu_Client`。在某些语言环境中，这个类是线程安全的，多个线程可以共享同一份实例，但在 C-SDK 中它被设计为线程不安全的。一个重要的原因是我们试图简化内存管理的负担。HTTP 请求结果的生命周期被设计成由`Qiniu_Client`负责，在下一次请求时会自动释放上一次 HTTP 请求的结果。这有点粗暴，但在多数场合是合理的。如果某个 HTTP 请求结果的数据需要长期使用，你应该复制一份。例如：

    @gist(gist/server.c#stat)

这个例子中，`Qiniu_RS_Stat`请求返回了`Qiniu_Error`和`Qiniu_RS_StatRet`两个结构体。其中的 `Qiniu_Error` 类型是这样的：

    @gist(../qiniu/base.c#error)

`Qiniu_RS_StatRet` 类型是这样的：

    @gitst(../qiniu/rs.c#statret)

值得注意的是，`Qiniu_Error.message`、`Qiniu_RS_StatRet.hash`、`Qiniu_RS_StatRet.mimeType` 都声明为 `const char*` 类型，是个只读字符串，并不管理字符串内容的生命周期。这些字符串什么时候失效？下次 `Qiniu_Client` 发生网络 API 请求时失效。如果你需要长久使用，应该复制一份，比如：

    hash = strdup(ret.hash);


<a name="error-handling"></a>

### 错误处理与调试

在 HTTP 请求出错的时候，C-SDK 统一返回了一个`Qiniu_Error`结构体：

    @gist(../qiniu/base.c#error)

即一个错误码和对应的读者友好的消息。这个错误码有可能是 cURL 的错误码，表示请求发送环节发生了意外，或者是一个 HTTP 错误码，表示请求发送正常，服务器端处理请求后返回了 HTTP 错误码。

如果一切正常，`code`应该是 200，即 HTTP 的 OK 状态码。如果不是 200，则需要对`code`的值进行相应分析。对于低于 200 的值，可以查看 [cURL 错误码](http://curl.haxx.se/libcurl/c/libcurl-errors.html)，否则应查看[七牛云存储错误码](http://docs.qiniutek.com/v2/api/code/)。

如果`message`指示的信息还不够友好，也可以尝试把整个 HTTP 返回包打印出来看看：

    @gist(gist/server.c#debug)


<a name="io-put"></a>

## 上传文件

为了尽可能地改善终端用户的上传体验，七牛云存储首创了客户端直传功能。一般云存储的上传流程是：

    客户端（终端用户） => 业务服务器 => 云存储服务

这样多了一次上传的流程，和本地存储相比，会相对慢一些。但七牛引入了客户端直传，将整个上传过程调整为：

    客户端（终端用户） => 七牛 => 业务服务器

客户端（终端用户）直接上传到七牛的服务器，通过DNS智能解析，七牛会选择到离终端用户最近的ISP服务商节点，速度会比本地存储快很多。文件上传成功以后，七牛的服务器使用回调功能，只需要将非常少的数据（比如Key）传给应用服务器，应用服务器进行保存即可。

<a name="io-put-flow"></a>

### 上传流程

在七牛云存储中，整个上传流程大体分为这样几步：

1. 业务服务器颁发 [uptoken（上传授权凭证）](http://docs.qiniutek.com/v3/api/io/#upload-token)给客户端（终端用户）
2. 客户端凭借 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token) 上传文件到七牛
3. 在七牛获得完整数据后，发起一个 HTTP 请求回调到业务服务器
4. 业务服务器保存相关信息，并返回一些信息给七牛
5. 七牛原封不动地将这些信息转发给客户端（终端用户）

需要注意的是，回调到业务服务器的过程是可选的，它取决于业务服务器颁发的 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token)。如果没有回调，七牛会返回一些标准的信息（比如文件的 hash）给客户端。如果上传发生在业务服务器，以上流程可以自然简化为：

1. 业务服务器生成 uptoken（不设置回调，自己回调到自己这里没有意义）
2. 凭借 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token) 上传文件到七牛
3. 善后工作，比如保存相关的一些信息

服务端生成 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token) 代码如下：

    @gist(gist/server.c#uptoken)

上传文件到七牛（通常是客户端完成，但也可以发生在服务端）：

    @gist(gist/client.c#upload)

<a name="io-put-policy"></a>

### 上传策略

[uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token) 实际上是用 AccessKey/SecretKey 进行数字签名的上传策略(`Qiniu_RS_PutPolicy`)，它控制则整个上传流程的行为。让我们快速过一遍你都能够决策啥：

* 一个 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token) 可以被用于多次上传（只要它还没有过期）。
* `scope` 限定客户端的权限。如果 `scope` 是 bucket，则客户端只能新增文件到指定的 bucket，不能修改文件。如果 `scope` 为 bucket:key，则客户端可以修改指定的文件。
* `callbackUrl` 设定业务服务器的回调地址，这样业务服务器才能感知到上传行为的发生。可选。
* `asyncOps` 可指定上传完成后，需要自动执行哪些数据处理。这是因为有些数据处理操作（比如音视频转码）比较慢，如果不进行预转可能第一次访问的时候效果不理想，预转可以很大程度改善这一点。
* `returnBody` 可调整返回给客户端的数据包（默认情况下七牛返回文件内容的 `hash`，也就是下载该文件时的 `etag`）。这只在没有 `callbackUrl` 时有效。
* `escape` 为真（非0）时，表示客户端传入的 `callbackParams` 中含有转义符。通过这个特性，可以很方便地把上传文件的某些元信息如 `fsize`（文件大小）、`imageInfo.width/height`（图片宽度/高度）、`exif`（图片EXIF信息）等传给业务服务器。
* `detectMime` 为真（非0）时，表示服务端忽略客户端传入的 `mimeType`，自己自行检测。

关于上传策略更完整的说明，请参考 [uptoken](http://docs.qiniutek.com/v3/api/io/#upload-token)。


<a name="resumable-io-put"></a>

### 断点续上传


<a name="io-get"></a>

## 下载文件

每个 bucket 都会绑定一个或多个域名（domain）。如果这个 bucket 是公开的，那么该 bucket 中的所有文件可以通过一个公开的下载 url 可以访问到：

    http://<domain>/<key>

假设某个 bucket 既绑定了七牛的二级域名，如 hello.qiniudn.com，也绑定了自定义域名（需要备案），如 hello.com。那么该 bucket 中 key 为 a/b/c.htm 的文件可以通过 http://hello.qiniudn.com/a/b/c.htm 或 http://hello.com/a/b/c.htm 中任意一个 url 进行访问。

<a name="io-get-private"></a>

### 下载私有文件

如果某个 bucket 是私有的，那么这个 bucket 中的所有文件只能通过一个的临时有效的 url 访问：

    http://<domain>/<key>?token=<dntoken>

其中 dntoken 是由业务服务器签发的一个[临时下载授权凭证](http://docs.qiniutek.com/v3/api/io/#private-download)。生成 dntoken 代码如下：

    @gist(gist/server.c#dntoken)

生成 dntoken 后，服务端可以下发 dntoken，也可以选择直接下发临时的 downloadUrl（推荐这种方式，看起来灵活性更好，避免了客户端自己组装 url）。客户端收到 downloadUrl 后，和公有资源类似，直接用任意的 HTTP 客户端就可以下载该资源了。唯一需要注意的是，在 downloadUrl 失效却还没有完成下载时，需要重新向服务器申请授权。

无论公有资源还是私有资源，下载过程中客户端并不需要七牛 C-SDK 参与其中。

<a name="io-https-get"></a>

### HTTPS 支持

如果你的资源希望支持 https 下载，则有如下限制：

1. 不能用 xxx.qiniudn.com 这样的二级域名，只能用 dn-xxx.qbox.me 域名。样例：https://dn-abc.qbox.me/1.txt
2. 使用自定义域名是付费的。我们并不建议使用自定义域名，但如确有需要，请联系我们的销售人员。

<a name="resumable-io-get"></a>

### 断点续下载

无论是公有资源还是私有资源，获得的下载 url 支持标准的 HTTP 断点续传协议。考虑到多数语言都有相应的断点续下载支持的成熟方法，七牛 C-SDK 并不提供断点续下载相关代码。


<a name="rs"></a>

## 资源操作

<a name="rs-stat"></a>

### 获取文件信息

<a name="rs-delete"></a>

### 删除文件

<a name="rs-copy-move"></a>

### 复制/移动文件

<a name="rs-batch"></a>

### 批量操作