---
sidebarDepth : 0
---

# 深入H5与NativeApp通信原理

[[TOC]]

Hybrid App，俗称混合应用，即混合了 Native技术 与 Web技术 进行开发的移动应用。在APP中使用 WebView 作为容器直接承载 Web页面。

## webview的介绍

webview是移动APP中展示网页的容器组件。

### IOS上的webview

![wv-ios.png](https://cdn.nlark.com/yuque/0/2020/png/424641/1584263311203-f46294e9-081b-4ad5-8a92-3e19d83aefad.png#align=left&display=inline&height=743&name=wv-ios.png&originHeight=743&originWidth=739&size=53857&status=done&style=none&width=739)

### 安卓上webview

|  | webkit for Webview | chromium for Webview |
| --- | --- | --- |
| 兼容 | Android4.4以下 | Android4.4以上 |
| JS引擎 | webCore javascript | V8 |
| H5元素支持情况 | 278 | 434 |
| 运行调试 | 不支持 | 支持 |
| 内存占用 | 小 | 大 |
| H5新API（Canvas,SVG,WebGL） | 不支持 | 支持 |



## 业界方案

- 1、基于 WebView UI 的基础方案，市面上大部分主流 App 都有采用，例如微信JS-SDK，通过 JSBridge 完成 H5 与 Native 的双向通讯，从而赋予H5一定程度的原生能力。

- 2、基于 Native UI 的方案，例如 React-Native、Weex。在赋予 H5 原生API能力的基础上，进一步通过 JSBridge 将js解析成的虚拟节点树(Virtual DOM)传递到 Native 并使用原生渲染。

- 3、近期比较流行的小程序方案，通过更加定制化的 JSBridge，并使用双 WebView 双线程的模式隔离了JS逻辑与UI渲染，形成了特殊的开发模式，加强了 H5 与 Native 混合程度，提高了页面性能及开发体验。

- 4、Flutter： 他是谷歌的移动UI框架，可以快速在iOS和Android上构建高质量的原生用户界面。采用了高性能渲染引擎(Skia)，界面开发语言使用dart，底层渲染引擎使用C, C++，框架层面针对IOS和安卓平台分别实现了对应的UI风格组件库。相比webview渲染、nativeUI渲染，性能更加出色。由于推出不久，性能还不够稳定。


## Hybrid技术原理

混合开发中最核心的点就是 Native端 与 H5端 之间的双向通讯层，其实这里也可以理解为我们需要一套跨语言通讯方案，来完成 Native(Java/Objective-c/...) 与 JavaScript 的通讯。这个方案就是我们所说的 JSBridge，而实现的关键，便是作为容器的 WebView，一切的原理都是基于 WebView 的机制。

### (一) JavaScript 调用 Native接口方式

**基于 WebView 的机制和开放的 API, 实现这个功能有三种常见的方案：**

- 1）、API注入，原理其实就是 Native 获取 JavaScript环境上下文，并直接在上面挂载对象或者方法，使 js 可以直接调用，Android 与 IOS 分别拥有对应的挂载方式。

- 2）、WebView 中的 prompt/confirm/alert 拦截，通常使用 prompt，因为这个方法在前端中使用频率低，比较不会出现冲突；

- 3）、WebView URL Scheme 跳转拦截；

- 4) 、在IOS上wkwebview上有window.webkit.messageHandler实现通信，支持IOS8及以上

#### 方法3 - URL拦截方案

##### 1、实现原理

 在 WebView 中发出的网络请求，客户端都能进行监听和捕获

##### 2、协议的定制

 我们需要制定一套URL Scheme规则，通常我们的请求会带有对应的协议开头，例如常见的 https://xxx.com 或者 file://1.jpg，代表着不同的含义。我们这里可以将协议类型的请求定制为:
 
> xxcommand://xxxx?param1=1&param2=2
 
- (1)、xxcommand:// 只是一种规则，可以根据业务进行制定，使其具有含义，例如我们定义 xxcommand:// 为公司所有App系通用，为通用工具协议：

> xxcommand://getProxy?a=a&b=2


而定义 xxapp:// 为每个App单独的业务协议。
	
> xxapp://openCamera?h=2
	
 **不同的协议头代表着不同的含义，这样便能清楚知道每个协议的适用范围。**
 
- (2)、 这里不要使用 location.href 发送，因为其自身机制有个问题是同时并发多次请求会被合并成为一次，导致协议被忽略，而并发协议其实是非常常见的功能。我们会使用创建 iframe 发送请求的方式。
 
- (3)、 通常考虑到安全性，需要在客户端中设置域名白名单或者限制，避免公司内部业务协议被第三方直接调用。
 
##### 3.协议的拦截
 
客户端可以通过 API 对 WebView 发出的请求进行拦截：
- IOS上: shouldStartLoadWithRequest
- Android: shouldOverrideUrlLoading

当解析到请求 URL 头为制定的协议时，便不发起对应的资源请求，而是解析参数，并进行相关功能或者方法的调用，完成协议功能的映射。

##### 4、协议回调

由于协议的本质其实是发送请求，这属于一个异步的过程，因此我们便需要处理对应的回调机制。这里我们采用的方式是JS的事件系统，这里我们会用到 window.addEventListener 和 window.dispatchEvent这两个基础API；

- 发送协议时，通过协议的唯一标识注册自定义事件，并将回调绑定到对应的事件上。

- 客户端完成对应的功能后，调用 Bridge 的dispatch API，直接携带 data 触发该协议的自定义事件。

原生的事件机制是我们非常熟悉的，例如我们常用的

```javascript
window.addEventListener('DOMContentLoaded', () => {});
```
通过事件的机制，会让开发更符合我们前端的习惯，例如当你需要监听客户端的通知时，同样只需要在通过 addEventListener 进行监听即可。

**Tips:** 这里有一点需要注意的是，应该避免事件的多次重复绑定，因此当唯一标识重置时，需要removeEventListener对应的事件。

##### 5、参数传递方式

由于 WebView 对 URL 会有长度的限制，因此常规的通过 search参数 进行传递的方式便具有一个问题，既 当需要传递的参数过长时，可能会导致被截断，例如传递base64或者传递大量数据时。‘’

因此我们需要制定新的参数传递规则，我们使用的是函数调用的方式。这里的原理主要是基于:

> Native 可以直接调用 JS 方法并直接获取函数的返回值。

我们只需要对每条协议标记一个唯一标识，并把参数存入参数池中，到时客户端再通过该唯一标识从参数池中获取对应的参数即可。

### (二) Native 调用 Javascript接口方式

由于 Native 可以算作 H5 的宿主，因此拥有更大的权限，上面也提到了 Native 可以通过 WebView API直接执行 Js 代码。这样的权限也就让这个方向的通讯变得十分的便捷。

- IOS: stringByEvaluatingJavaScriptFromString

```swift
// Swift
webview.stringByEvaluatingJavaScriptFromString("alert('NativeCall')")
```

- Android: loadUrl (4.4-)

```java
// 调用js中的JSBridge.trigger方法
// 该方法的弊端是无法获取函数返回值；
webView.loadUrl("javascript:JSBridge.trigger('NativeCall')")
```
**Tips:** 当系统低于4.4时，evaluateJavascript 是无法使用的，因此单纯的使用 loadUrl 无法获取 JS 返回值，这时我们需要使用前面提到的 prompt 的方法进行兼容，让 H5端 通过 prompt 进行数据的发送，客户端进行拦截并获取数据。

- Android: evaluateJavascript (4.4+)

```java
// 4.4+后使用该方法便可调用并获取函数返回值；
mWebView.evaluateJavascript（"javascript:JSBridge.trigger('NativeCall')",      new ValueCallback<String>() {
    @Override
    public void onReceiveValue(String value) {
        //此处为 js 返回的结果
    }
});
```

基于上面的原理，我们已经明白 JSBridge 最基础的原理，并且能实现 Native <=> H5 的双向通讯机制了。
![](http://doc.stip.bbkedu.com/uploads/a3b10e5359e0e91291c6feb07ac5eafc.jpg)

## JSBridge 的接入

接下来，我们来理下代码上需要的资源。实现这套方案，从上图可以看出，其实可以分为两个部分:

#### JS部分(bridge): 在JS环境中注入 bridge 的实现代码，包含了协议的拼装/发送/参数池/回调池等一些基础功能。

#### Native部分(SDK)：在客户端中 bridge 的功能映射代码，实现了URL拦截与解析/环境信息的注入/通用功能映射等功能。

我们这里的做法是，将这两部分一起封装成一个 Native SDK，由客户端统一引入。客户端在初始化一个 WebView 打开页面时，如果页面地址在白名单中，会直接在 HTML 的头部注入对应的 bridge.js。这样的做法有以下的好处：

#### 双方的代码统一维护，避免出现版本分裂的情况。有更新时，只要由客户端更新SDK即可，不会出现版本兼容的问题；

- App的接入十分方便，只需要按文档接入最新版本的SDK，即可直接运行整套Hybrid方案，便于在多个App中快速的落地；

- H5端无需关注，这样有利于将 bridge 开放给第三方页面使用。


这里有一点需要注意的是，协议的调用，一定是需要确保执行在bridge.js 成功注入后。由于客户端的注入行为属于一个附加的异步行为，从H5方很难去捕捉准确的完成时机，因此这里需要通过客户端监听页面完成后，基于上面的回调机制通知 H5端，页面中即可通过window.addEventListener('bridgeReady', e => {})进行初始化。


## App中 H5 的接入方式

将 H5 接入 App 中通常有两种方式：

### (1) 在线H5，这是最常见的一种方式。我们只需要将H5代码部署到服务器上，只要把对应的 URL地址 给到客户端，用 WebView 打开该URL，即可嵌入。该方式的好处在于:

- 独立性强，有非常独立的开发/调试/更新/上线能力；
- 资源放在服务器上，完全不会影响客户端的包体积；
- 接入成本很低，完全的热更新机制。

但相对的，这种方式也有对应的缺点:

- 完全的网络依赖，在离线的情况下无法打开页面；
- 首屏加载速度依赖于网络，网络较慢时，首屏加载也较慢；

通常，这种方式更适用在一些比较轻量级的页面上，例如一些帮助页、提示页、使用攻略等页面。这些页面的特点是功能性不强，不太需要复杂的功能协议，且不需要离线使用。在一些第三方页面接入上，也会使用这种方式，例如我们的页面调用微信JS-SDK。

### (2) 内置包H5，这是一种本地化的嵌入方式，我们需要将代码进行打包后下发到客户端，并由客户端直接解压到本地储存中。通常我们运用在一些比较大和比较重要的模块上。其优点是:

- 由于其本地化，首屏加载速度快，用户体验更为接近原生；
- 可以不依赖网络，离线运行；

但同时，它的劣势也十分明显:

- 开发流程/更新机制复杂化，需要客户端，甚至服务端的共同协作；
- 会相应的增加 App 包体积；

这两种接入方式均有自己的优缺点，应该根据不同场景进行选择。





