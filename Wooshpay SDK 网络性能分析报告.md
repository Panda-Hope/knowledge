## 说明：
本次报告用于分析用户所反馈的关于Wooshpay SDK渲染速度缓慢的问题，逐帧分析网络链路加载的各个模块来说明耗时消费过长的问题。

## 总览：
首先先看下首屏渲染耗时统计：

<img width="800" src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/%E6%88%AA%E5%B1%8F2022-06-27%2016.34.38.png" />


经过多次测试，目前来讲平均首屏渲染耗时在2S左右，网速较差或者3G、4G网络情况下，渲染耗时甚至可能到4~5S，总体用户体验感较差。

其次我们来看下存在网络缓存的情况下的首屏渲染耗时：

<img width="800" src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_17-19-38.png" />


在这里发现当存在用户数据缓存的情况(即用户重新刷新页面)下，首屏渲染耗时被大幅减少到平均600~700ms左右，说明整个页面渲染耗时较长的主要原因，

更多在于网络连接耗时尤其是TCP连接请求耗时。接下来我们分析每一个网络节点的消耗时长，来分析每一个具体的环节：

## 链路节点：
首先我们将整个SDK的网络请求链路一一列出：

- 用户网页渲染(这里是Wooshpay Demo自身)
- 加载Wooshpay SDK
- 向Wooshpay SDK Server发出收银台渲染请求
- 通过Iframe加载Wooshpay 收银台页面
- 异步加载Wooshpay 收银台CSS与JS，完成收银台首屏渲染
首先，我们思考一下整个收银台页面渲染链路其本身，核对收银台渲染交互逻辑，确认收银台渲染链路其自身并无逻辑问题~，然后开始核对整个链路上面的每个环节。

### 用户网页渲染：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_17-55-2.png" width=400 />

用户网页渲染速度由客户自身决定，Wooshpay SDK不参与此链路，但由于本次实验的客户对象本身即为Wooshpay SDK Demo，这里我们仍将其渲染速度加以列出。

可以看到网络的第一个请求耗时相当之长，足足有1.3s。

### Wooshpay SDK加载：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_19-11-16.png" width=400 />

到了下载Wooshpay SDK Js时，可以明显看到速度有所提升，由于HTTP1.0协议默认会为每一个请求构建一个新的TCP连接，

但是DNS解析并不会重复获取，由此我们知悉，第一次请求速度缓慢的主要原因在于域名的DNS解析缓慢。

### 请求开始渲染收银台：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_19-20-42.png" width=400 />

在收银台请求环节，我们可以看到网络请求仍然是主要耗时原因，但是同样可以看到收银台请求渲染的335ms渲染速度是优于SDK加载的665ms的，

其主要原因在于收银台请求的TCP包体制大小明显小于SDK TCP包的体积大小，因此压缩请求资源体积大小同样是很重要的一部分。

### 加载收银台渲染页面：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_19-35-54.png" width=400 />


到了收银台加载页面，我们开始看到渲染速度又开始变得缓慢起来，但是收银台页面本事的体积大小却并没有多大，

那么在此时为什么网络请求又开始变得缓慢了呢？，在经过一段时间的查询与搜索后，我发现浏览器会优先加载本身的JS、CSS资源，而放慢iframe的加载，

但是这里我发现了一个很细节的小技巧，浏览器会优先加载存在内容的iframe，因此此时我们可以将iframe设置css属性并提前设置好iframe链接地址，同时将外联JS放置在最后来提升iframe的加载优先级。

```code
<html>
<head> </head>
<body>
<h1>Main Page: Example 2</h1>

<iframe id="myiframe" src="https://js.wooshpay.com" width="90%" height="90%" style="margin:auto;display:block"> </iframe>
<script src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.6.5/angular.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/ng2-bootstrap/1.6.2/ng2-bootstrap.umd.js"></script>
<h1>Footer</h1>
</body>
</html>
```

加载CSS、JS开始渲染收银台：

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/image2022-6-27_19-51-55.png" width="400" />

到了收银台页面，我们看到由于iframe已经完成加载，此时的渲染速度又开始再次慢了下来，但由于TCP请求本身耗时较长，

此时仍旧有500ms+的渲染时长，而由于这一步是整个收银台渲染的最后一步，完成了这一步才能够最终让用户获取到收银台页面的UI感知，

因此这里准备一个骨架屏预加载效果，用于提前为用户获取UI感知。


总结：
首先，域名的DNS解析速度缓慢，导致请求无法在第一时间得到响应，这里可以部署DNS解析加速
其次，TCP连接请求往往在500ms以上，而且请求资源本身较为简单，超过了一般服务器响应时间，说明服务器带宽和响应速度较为缓慢
第三，对于网络资源包的体积也需要压缩以减少不必要的请求耗时，同时对于静态资源也可以部署CDN加速以提升下载速度
第四，对于iframe的加载需要提前设置好加载链接，以减少非必要的等待耗时，同时提升iframe的加载优先级
第五，增加骨架屏效果，以在完成实际收银台页面渲染之前，及时给到用户一个UI反馈，提升用户体验
备注：
本次链路分析的每个环节所得数据均由多次实验而得的平均数据，数据本身较为符合真实用户访问速度，

但由于业务范围全球化，对于不同地区用户所能够得到的体验，本次实验仅持有参考意见，不代表全部用户体验结果。

附件：
Wooshpay SDK 网络分析报告
