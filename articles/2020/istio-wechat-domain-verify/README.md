
# Istio 下微信业务域名验证失败的解决办法

最近做的项目已经使用 Istio 作为所有服务的边界入口网关（ingress gateway），可以很灵活地将流量路由到不同的服务，认证授权，部署策略，服务观测等功能，体验非常棒。

但是有一个小问题一直困扰，由于业务依赖于微信API，它需要对业务域名进行认证。方案就是提供一个文本文件，让你放在域名的根路径下，然后他去请求这个文件。看起来挺简单的，只需要在 virtual service 里进行如下的配置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: mp
spec:
  gateways:
  - my-gateway
  hosts:
  - example.com
  http:
  - name: wx-domain-verify
    match:
    - uri:
        exact: /2746855935.txt
    rewrite:
      uri: /wx-domain-files/2746855935.txt
    route:
    - destination:
        host: file-server.public.svc.cluster.local
        port:
          number: 80
  - route:
    - destination:
        host: wechat-server.mp.svc.cluster.local
        port:
          number: 80 # gRPC server
```

这里我们使用的方式是额外部署了一个nginx文件服务，当匹配到文件根路径时，对url进行重写然后将请求分发到文件服务器即可。当然重定向/转发到一个对象存储的CDN服务可能更方便一些。

然后使用curl测试也确实没毛病，奇怪就奇怪在提交到微信老是验证失败，对比了先前裸用nginx成功的案例也没啥毛病，要说区别，也仅仅是返回的响应头稍稍不同，nginx使用`Content-Type`这样首字母大写的命名，istio使用`content-type`，但实际上在[RFC规范](https://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html)中，**消息头部的字段名是大小写不敏感**的。如果看过http2的响应，你会发现已经全部都小写了，就算在代码中是大写。建议以后全小写吧，好看点。

```http
$ curl -i --http1.1 https://example.com/2746855935.txt
HTTP/1.1 200 OK
server: istio-envoy
date: Wed, 04 May 2020 18:39:30 GMT
content-type: text/plain
content-length: 33
last-modified: Tue, 28 Apr 2020 09:45:37 GMT
etag: "5ea7fb41-21"
accept-ranges: bytes
x-envoy-upstream-service-time: 1

d0f5c62a4de47a289ad9f7bec1cca5a5

$ curl -i https://example1.com/IE2GtO4OUe.txt
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Wed, 04 May 2020 18:37:41 GMT
Content-Type: text/plain
Content-Length: 33
Last-Modified: Tue, 28 Apr 2020 04:04:54 GMT
Connection: keep-alive
ETag: "5ea7ab66-21"
Accept-Ranges: bytes

fd82761c9f79e33e9627b1cf9133fa61
```

排除上述原因后，打开了istio ingress gateway的访问日志，突然发现一个奇怪的状态码，426，还是头一次看到，它的[含义](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/426)是服务器认为客户端所使用的HTTP协议版本过低，要求其升级版本，然后再看到请求的版本是`HTTP/1.0`也就豁然开朗了。有趣的是，解决这个问题的时间是4月29号，刚好是最大的客户端错误状态码，哈哈哈。

```
[2020-04-29T06:54:33.815Z] "GET /2746855935.txt HTTP/1.0" 426 - "-" "-" 0 0 0 - "-" "Mozilla/4.0" "-" "mp.example.com" "-" - - 10.244.0.5:80 10.244.0.1:50048 - -
```

微信使用的1.0的协议去请求这个域名验证文本文件，然后istio，或者说envoy代理服务器默认最低版本是1.1，所以才导致了这个结果。curl，以及现在绝大多数http客户端，使用的至少都是1.1，很少看到1.0的了，但是出于兼容性，像nginx还是支持1.0及以上的。

解决办法也有，envoy在它的api中提供了[开启1.0支持的选项](https://www.envoyproxy.io/docs/envoy/v1.13.1/api-v3/config/core/v3/protocol.proto#envoy-v3-api-msg-config-core-v3-http1protocoloptions)，由于istio的CRD没有暴露这个选项，最好的方式是使用它提供的 [Envoy Filter API](https://istio.io/docs/reference/config/networking/envoy-filter/) 直接修改envoy的配置，具体如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: accept-http10
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: NETWORK_FILTER # http connection manager is a filter in Envoy
    match:
      # if context omitted then this applies to both sidecars and gateways
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          http_protocol_options:
            #header_key_format:
            #  proper_case_words: {}
            accept_http_10: true
```

这段配置将最后`value`的部分merge到了当前的ingress gateway中。取消注释还可以启用首字母大写的头命名风格，但不推荐。

```http
$ curl -i --http1.0 https://example.com/2746855935.txt
HTTP/1.0 200 OK
server: istio-envoy
date: Wed, 04 May 2020 18:26:01 GMT
content-type: text/plain
content-length: 33
last-modified: Tue, 28 Apr 2020 09:45:37 GMT
etag: "5ea7fb41-21"
accept-ranges: bytes
x-envoy-upstream-service-time: 3
connection: close

d0f5c62a4de47a289ad9f7bec1cca5a5
```

OK，问题解决。下面谈谈从这个问题中的一些思考。

首先是HTTP不同版本之间的区别，网络协议有一个很重要的特点就是向下保持兼容，所以支持h2的服务器，铁定是支持http1.1的，当然istio默认不支持http1.0也无可厚非，毕竟1.0的RFC是1996年发布的，1.1的草案也在97年就发布了，直到14年h2才发布。到目前为止，http1.1仍然是主流并且是兼容性最好的。

每次升级版本都是对上一版本的存在的问题进行改进，比如1.0默认是不支持tcp连接复用的，每个请求都会经历tcp三次握手和四次挥手的阶段，对于存在比较多的请求的页面延迟是无法接受的；所以http1.1以keep-alive的方式保持tcp连接，一段时间没有http请求才会超时断开；但是随着web的发展，对于像腾讯首页这样，可能有上百个请求，延迟还是很高，单个域名的tcp连接数也有限制，所以h2通过单个连接的多路复用并发交错发送请求流，最大化的利用的tcp双向流的特性，可以说是将tcp的性能发挥到极致了；但是tcp固有的瓶颈（队首阻塞，拥塞控制等），仍然无法满足今后的web快速发展，所以基于udp的http3草案一直在推进，甚至一些公司已经开始使用了。

当然每个版本还有很多很多特性，这里仅仅就连接进行对比，这也反应了不同的年代我们使用web状况的对比。微信使用1.0我觉得可能有下面几个原因吧：

-   web兼容性，考虑到有一些用户的服务器确实使用1.0的版本且无法升级
-   优化连接资源，fire then close，考虑到这种基本的文件下载场景，连接复用没有意义
-   使用自研的http客户端，比较简单的实现，缺少robust

但是我认为这三个原因都不足以成为理由。首先微信早早就要求小程序的开发者的API接口使用HTTPS，TLS的接受程度也就最近10年的事情，比1.0要晚得多，这说不过去。其次，使用http1.1时，在connection头部添加close同样可以做到请求返回后直接关闭tcp连接。最后，在调用http请求时，不能简单地认为请求只有一次，比如响应有301重定向还是比较常见的，客户端需要处理有关的状态码，在使用http1.0时，需要考虑服务器不支持返回426的情况下，升级协议发起新的请求。所以别看是一个简单的需求，面对复杂的外部环境，要写出健壮的程序还是比较有挑战的。

所以我觉得比较好的方式是，发起http1.1请求，该业务场景下客户端的连接不需要管理，当服务器返回`505 HTTP Version Not Supported`时，再降级为1.0，毕竟现在只支持1.0的服务应该相当少了。从软件设计的层面来说，也应该是这样：

> Interfaces should be designed to make the most common usage as simple as possible
> 
> 摘录来自: John Ousterhout. “A Philosophy of Software Design。” Apple Books.

最后，希望微信能修改一下[校验文件检查失败自查指引](https://developers.weixin.qq.com/community/develop/doc/00084a350b426099ab46e0e1a50004)，告知用户使用HTTP版本，否则，可能就会出现我们的问题，这些指引都没有问题，但还是过不了。

最近浪迹于Istio，envoy，gRPC中，发现很多有趣很新的东西，接下来会陆续写一些文章，敬请期待！

## EOF

```yaml
summary: Istio 下微信业务域名验证失败的解决办法
weather: hot
license: cc-40-by
location: mars
background: istio.png
tags: [istio, envoy, http, network, wechat]
date: 2020-05-07T01:00:14+08:00
```
