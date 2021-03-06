
# Istio Architecture

疫情期间，天天宅家，看书的时间也多了起来，花了周末的时间把《Istio: Up and Running》给过了。

上周（3月5号），Istio发布了1.5版本。在部署易用性还有性能上，看起来都比我们正在使用的1.3.x版本要好不少。手头上恰好也有这本书，本着温故知新（折腾）的态度，本地下载安装了最新版，对照着官方文档和这本书，快速地过了一遍。对Istio的架构组件，配置细节和一些内部流程都有了进一步认识。

需要注意的是，istio自身的迭代比较快，而这本书所讲授的版本比较低，不少使用配置（CRD）上已经发生了变化。所以比较推荐阅读方式是，通过官方文档动手实践，而在这本书里着重去理解什么是Service Mesh，为什么需要它，以及具体到Istio中各个组件是怎么工作的。设计思路比怎么使用更重要。

下面也就以自己的理解来做一个读书笔记，或者回顾吧。

## 为什么需要Istio？

先从10年前说起吧。

最开始我们是直接在物理机上通过脚本部署维护服务的，随着需求变更版本迭代，机器上会发生很多变化，如动态库，配置文件，版本等，出了问题难以维护和调试。

既然会发生变化，那么我们把所有的东西都打包到虚拟机上，不同的服务就彻底的隔离了，应用看到的东西就不会发生变化了。

但是虚拟机对于开发同学来说太重了，试想一下，开发环境上配置一堆的虚拟机？于是Docker出来解决了这个问题，基于进程的模型以及应用和文件系统一起打包分发，解耦了应用和机器。从此妈妈再也不用担心我的运行时环境了，贼开心。

但是docker并没有解决所有的问题，我可能有很多docker容器要跑，东西多了就需要管理；另外，docker最终还是需要运行在机器上的，怎么合理的利用网络，计算和存储资源？看起来是不是很像操作系统的职责？于是服务器的操作系统，Kubernetes接管了容器编排这个领域。它解决了这些问题，并且将整个PaaS上下游串起来了，依托k8s建立的CNCF已经形成了强大的生态。

看起来k8s解决了所有的问题？在基础设施方面应该是，但大多架构师/开发者关心的业务架构，随着微服务的流行，似乎也没有提供好办法（自带的Ingress太弱了）。

什么是微服务，我觉得是业务发展到一定程度，社会分工下的必然产物吧。区别于原始的所有的业务代码集中在一起打包成一个大的单体应用，微服务按照一些方式（业务，架构，需求，不同的部门等）将大的服务拆分为可以各自独立开发部署的若干个服务，这些服务可以运行在不同的机器上，运行的实例可能也有多个，相互之间通过网络通信。这样，鸡蛋不用都放在一个篮子里，一个服务挂了，另一个还OK，并且根据不同服务的重要性去QoS等。开发和部署都敏捷了，老板贼开心。但这世界是守恒的，增加一些东西就得放弃一些东西。

服务与服务间的通信，可以看作是一张虚拟的网，这也是服务网格（Service Mesh）这名字的由来吧。网状结构的东西通常都不简单，更不用说节点一多了，怎么管理？简单来说，主要有三个问题：流量控制，服务间的网络怎么路由到彼此？服务之间的访问控制，A能访问B，那C能访问A吗？我想观察服务之间的调用路径，指标监控还有访问日志？

Istio就是来解决这几个问题的。按照官网它的设计目标有：流量最大的透明化，网络很复杂，不应该把这复杂性带到应用程序中；可扩展性，成功开源项目标配；平台可移植性，这个没有意义，基本上就是集成到k8s中；策略一致性？不太明白，看起来应该是通过类似k8s那种声明式的API去描述和控制服务网格吧（使用体会）。

这里说一下自己的感受，我觉得流量最大透明化是Istio最核心的设计。软件设计的本质是降低复杂度，而软件最终是跑在硬件上，机器能提供什么？自上而下，网络，计算和存储。作为开发，我们主要关注的是计算（业务逻辑），存储通常会交给一堆的中间件（数据库，缓存，对象存储等），因为太复杂了，网络同样如此（想一下OSI七层模型，服务发现，负载均衡等）。网络和存储尽可能地对计算透明，这样计算就成了无状态的，无状态的东西好啊，水平扩展舒服。Istio这设计妙在，解决刚才说的那几个问题，不需要变更应用的代码（功能满足不了你不算）。

说了那么多，现在进入Istio正题。这里主要基于1.5的来介绍。

## 架构

![img](https://istio.io/docs/ops/deployment/architecture/arch.svg "Istio 1.5 架构")

Istio的服务网格主要分为控制（Control Plane）和数据（Data Plant）平面。数据平面由一系列包含在Pod中的Envoy Sidecar容器组成，而控制平面主要是由Gallery，Pilot，Citadel组成的Istiod容器。Istio1.5一个主要的变更就是移除了Mixer，将原来不同的组件集成到了一块，在功能上并没有变化。这里倒有种合久必分，分久必合的感觉，和我们说的拆分服务背道而驰？在易用性和性能上，这个取舍是可以的，再说对于使用者而言，也没增加复杂性。

### Envoy

先说数据平面吧，这里的关键就是Pod里的Envoy Sidecar容器。简单来说，Pod的出入站流量都被Envoy（istio-proxy）给代理了，所有的流量都经过它。在这个过程中，Envoy就能做很多事情，比如负载均衡，mTLS，L7/L4路由，熔断，重试等，可以说，Istio很多功能都是围绕着Envoy来的，通过控制平面抽象出规则配置，然后控制Envoy。

Envoy是一个C++写的新一代高性能代理服务器，这里不做多的介绍。只是说一下它是如何接管流量以及如何与控制平面交互的。

首先，Sidecar容器是对Pod里其他容器的增强，同一个Pod里所有的容器共享同一个网络命名空间，使得他们可以通过localhost通信。在Pod启动前，init容器会通过iptables设置规则，所有进出这个网络的流量都nat到envoy，按规则处理后发到应用容器（或者丢弃报错什么的）。

关于Envoy是怎么注入的后面会提到，下面说下Envoy的核心配置结构，从上到下：

-   Listeners 端口监听
-   Routers L4/L7路由
-   Clusters 服务发现
-   Endpoints 服务的一个地址信息，比如ip:port

我们可以手写静态配置文件，也可以通过Envoy提供的API（Discovery Service），分别是LDS，RDS，CDS，EDS去动态控制，这也是Envoy从众多代理服务器脱颖而出成为服务网格首选的代理Sidecar原因吧。

具体流程是，Envoy容器是一个包含envoy自身和一个agent（pilot-agent）的多进程容器，启动后会通过agent进程通过基于双向流的gRPC去和Istio的控制平面通信（Pilot），进可推，退可拉，最后将配置信息应用到envoy中。实际实现则是通过ADS聚合服务将多个XDS复用一个gRPC连接通信，提高了性能。需要注意的是，配置写入envoy使用的是最终一致性，所以配置生效需要时间传播。明白了这些，对于定位问题应该很有帮助，比如agent和pilot的连接是否OK，envoy的配置和当前状态？

接下来到控制平面瞧瞧。

### 安全策略

先说说Istio是如何解决安全问题吧，具体来说，对于两个通信服务：使用tls加密流量，身份验证（authn），权限检查（authz）。Istio通过Citadel调用Envoy的SDS（Security Discovery API）来做的安全控制。

Istio通过类似TLS的证书来确认身份。Citadel做为CA负责证书的签发和认证。具体来说，Envoy启动后，由agent创建私钥并提供Pod的信息ip，service account，名字等向Citadel发起CSR请求（gRPC），处理无误后下发证书给agent最后envoy就有了证书，在通信的时候就可以通过TLS，mTLS来确认对方身份，这就是istio1.5中服务间的身份认证，PeerAuthentication。对于终端用户（RequestAuthentication）则采用JWT/JWK来确认。

确认你是谁之后下一步就是判断你能做什么，这就是授权。Istio1.5通过一系列的声明式的策略（AuthorizationPolicy）来做实现。具体来说

-   from 请求的来源
-   to 想做什么
-   when 其他的条件约束
-   action 拒绝或放行

可以充分利用L4/L7协议中的各种属性来做判定，如请求头，http方法，url，端口等，相当灵活。

### 流量控制

下面来说说Istio中网络管理的核心组件，Pilot。它了解整个服务网格的拓扑结构，提供Envoy容器服务发现和流量管理的功能。简单来说，根据平台（k8s）workloads的变化和用户提交的配置文件，计算出每个Envoy所需要配置然后分发下去，过程在前面已经说了。下面我们具体来谈谈实际应用中配置间的关系和流程细节。

Istio抽象了一系列配置来做流量管理，最常见的有Gateway，Virtual Service，Destination Rule，Service Entry，他们都是通过名字关联在一起。

Gateway作为边界网关，代理整个网格的出入流量，入口网关叫做Ingress Gateway，也就是我们常说的反向代理网关，出口网关（Egress）可能没那么用的多，但原理是一样的。需要注意的是这两个网关并不是sidecar容器，对应着Envoy的Listeners。提供L4/L7协议的监听，TLS的剥离或添加，以及SNI透传，HTTPS重定向等功能。

Virtual Service主要用来做路由分发，流量只是逻辑上会经过它，但实际上没有。通过VS和Gateway的绑定，host匹配到的流量会经过VS的一系列的路由匹配，比如L4的端口，SNI或者L7的header，uri等等。需要注意的是，pilot会默认给每个服务匹配了一个VS，所以服务A访问服务B时，流量逻辑上经过了Gateway，VS，服务B，这个特殊的Gateway叫做mesh gateway，它监听着网格中所有服务的端口和host（\*），如果VS配置中没有绑定gateway，那它就默认地绑定了mesh gateway，换句话说，所有的流量都会和VS的host来匹配。这也解释了为什么服务间没有配置任何网络策略，也能够工作。

VS路由如果匹配不了就会返回404（L7）或者connect refuse（L4），匹配到了会继续下一个虚拟的路径，Destination Rule。这个主要是站在服务端的角度，强制要求请求方的流量做一些处理，比如是否需要TLS，或者双向TLS认证，负载均衡策略和熔断等。区别于超时，重试是在VS中配置的，我觉得区别主要是从客户端角度出发吧，所以才将VS和DS分开。需要注意的是DS不是必须的，对于没有定义的DS，Istio会使用默认的策略。

确定了目的地后，最后一步就是访问目标服务，Istio通过Service Entry来定义一个服务。服务就是网格中的真实存在的服务（k8s Service）或者用户自定义的服务（通常是网格外部服务）。需要注意的是前者是Istio内部维护的，我们无法看到和操作。通常我们用SE来定义一个外部服务（比如微信API调用），让他看起来好像这个服务就在网格中一样，这样更安全可控。一个服务包含了一组ip:port，或者通过DNS指向的下一个服务。服务可以是虚拟的，压根不存在，一个比较有趣的用法是用两个SE将流量经过egress gateway出口，虚拟路径可以认为是app->SE1（虚拟）->egress->SE2（真实），这样物理路径实际只有2跳。对于出口网格，还有一个概念值得注意，TLS originate，区别于ingress的tls termination，出口流量可以使用http，然后利用已有的经验，比如VS进行路由，甚至修改请求，然后tls封包，发送https请求到外部，这样一来，性能和可观测就都有了。

那么，还有一个问题，Istio的服务发现又是从哪来呢？答案是Gallery。

### Gallery

作为一个中间层，负责和底层平台（k8s）打交道，将平台的信息转换成Istio组件需要的格式。虽然没看过源码，但是大概率它就是一个k8s的Operator，watch k8s各种资源的变化，同步到Istio中来做对应的处理。比如一个服务的Pod创建了，Gallery就可以将这个Pod的信息，如ip，端口告诉Pilot添加到对应的Service Entry里。此外，还有一个关键Envoy Sidecar的注入，是通过k8s的Dynamic Admission Controller，当pod资源被提交到api-server后，根据策略修改并验证yaml，添加envoy sidecar容器到pod配置中，这也是Gallery的职责。

终于到最后一个问题了，服务间的访问控制和可观察性是怎么做到的？

### 可观察性

![img](https://archive.istio.io/v1.4/docs/ops/deployment/architecture/arch.svg "istio 1.4 架构")

在Istio1.5前，主要是通过mixer组件来做，mixer可以认为提供了两个接口，check和report，做成了两个pod（policy，telemetry），每次envoy发送请求都去mixer那check一遍，然后report观测信息给mixer，mixer加工后发给Prometheus。Mixer提供了进程内/外的adapter，让使用者去扩展。

由于流量每次都需要增加一跳到mixer，势必对性能有很大的影响，以及部署方面带来的麻烦，1.5终于废弃了mixer，转而将这些操作提到了envoy中。简单来说认证/授权相关的policy通过pilot下发；telemetry这个很有意思，在两个Envoy之间实现了一个协议，istio-peer-exchange，通过TLS握手后的ALPN协商交换metrics，后面这些数据就收集到Prometheus中了。还是Envoy牛逼啊。

没了mixer，想要扩展怎么办呢？istio1.5给了一个新的方案，基于WASM，看了[介绍](https://opensource.googleblog.com/2020/03/webassembly-brings-extensibility-to.html)很牛逼，感兴趣的同学可以了解一下。

一下子写了这么多，其实对Istio也是一个刚刚入门的状态，有些地方写得比较简单，如有错误的地方，请不要客气指出。

## EOF

```yaml
summary: istio 架构介绍
weather: fine
license: cc-40-by
location: 22, 114
background: ./lrg.jpg
tags: [k8s, istio, envoy, service mesh]
date: 2020-03-16T19:06:03+08:00
```
