
# 浅谈网络代理和 Kubernetes 网络模型

作为这一系列的第二篇，今天来谈谈作者比较感兴趣的一个主题，网络代理。

代理可以类比是现实生活中的各种中介吧，网络代理则与我们日常网上活动息息相关。客户端不是直接和服务端通信，而是经过中间的一个代理服务器的转发处理，这中间相当有一个用户定义的函数可以对进出的流量干点什么，一个网络请求在传输中很可能不只经过一个代理服务器。

![img](proxy.png "proxy")

很难想象没有代理，我们的网络会是什么样子，隐私，安全，性能，以及绕过防火墙，甚至于构建大规模的分布式系统。

限于时间和篇幅，本文介绍一些理论的东西，下一篇会实践一些这方面的有用且有趣的东西。

OK，说了这么多，先粗浅地聊聊网络模型。

我们知道，网络的实现TCP/IP是四层的，而在设计上则通常使用[OSI七层模型](https://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B)描述。七层模型自顶向下，功能及其常见协议简要如下：

| layer | function     | protocols       |
|----- |------------ |--------------- |
| 应用层 | 某类应用具体的业务逻辑 | http，ssh, mysql |
| 表示层 | 转换应用层的数据，如加密 | tls             |
| 会话层 | 会话管理     | socks5          |
| 传输层 | 两个主机间的数据传输 | tcp, udp        |
| 网络层 | 数据包在不同网络间的移动 | ip, icmp, igmp  |
| 链路层 | 直连节点间数据交换 | 以太网，Wi-Fi，蓝牙 |
| 物理层 | 网络设备和物理传输介质 | NaN             |

上层的数据向前追加下一层的协议信息封帧，一路向南，最后经由物理传输介质（比如光缆），这样数字信息就在物理世界中传输了。而我们今天谈论的主角，网络代理理论上可以工作在任一层（物理层目测不行），但是按照功能和通用性，常见的代理工作在L2/3/4/7，这也是TCP/IP在七层中的对应。越往下，代理提供的功能越单一，毕竟底层了解的信息相对较少。按照功能，有时候也叫负载均衡，本文统一称之为代理。

L2常见的代理是[网桥](https://zh.wikipedia.org/wiki/%E6%A9%8B%E6%8E%A5%E5%99%A8)，或者[交换机](https://zh.wikipedia.org/wiki/%E7%B6%B2%E8%B7%AF%E4%BA%A4%E6%8F%9B%E5%99%A8)，不同主机的网卡连接到同一个网桥上，主机间就可以直接连通了，网桥就是实实在在的一个代理，我们可以网桥内部增加一些代码，比如根据根据MAC的地址规则进行转发（[ARP Proxy](https://en.wikipedia.org/wiki/Proxy_ARP)）或者广播隔离（[VLAN](https://zh.wikipedia.org/wiki/%E8%99%9A%E6%8B%9F%E5%B1%80%E5%9F%9F%E7%BD%91)），但这样的功能局限在局域网，通常会配合L3一同工作。Kubernetes的flannel网络插件便是采用这种方式集群里所有的节点和容器连通（Overlay Network）。每个节点上的Pod都有一个虚拟网卡（veth）并连接到该节点上的同一个网桥（cni0）中，Pod里数据包统一通过网桥发送到Fannel.1（[VTEP](https://zh.wikipedia.org/wiki/%E8%99%9B%E6%93%AC%E5%B1%80%E5%9F%9F%E7%B6%B2%E6%93%B4%E5%B1%95)设备）中，通过VXLAN内核模块和etcd里存储的目的MAC/IP映射信息，将数据包封帧为一个UDP数据包，透过宿主机的网卡发往目的主机的Pod。看起来像是所有的节点和容器都是2层连通的。

![img](l2-fannel.png "layer2 proxy")

L3常见的代理是路由器，用来在不同的网络间转发数据包，因此路由器通常放置于两个网络的边界相互连接。L3最重要的信息便是源和目的IP地址，有了这俩信息，我们可以就可以根据路由策略把原本是A到B的转发到C上（DNAT），或者将源地址改成B（SNAT），甚至丢弃这个数据包（防火墙）。对比2层代理，3层适用范围就大多了。家里多个设备连接到路由器都可以上网（通常配合L4的NAPT使用），就使用[NAT](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)技术，这极大地缓解了IPv4地址短缺的问题（某种意义上也一再推迟了IPv6的部署）。一个比较实用的例子就是透明代理（TPPROXY），比如在路由器对非大陆的目的IP统统转发到一个中间无障碍代理服务器，这样就加速了境外网站的访问，而这对客户端是透明的，完全不需要做什么配置（当然路由配置还是要的），也意识不到其实已经走了代理。Kubernetes也有对应3层网络方案，如calico，每一个Pod都有一个虚拟IP，当节点A的Pod（IP1）想访问节点B的Pod（IP2）时，通过cni插件设置的路由规则，将其发送到某个中间路由设备上，最后到达节点B，看起来所有的节点和Pod都是3层连通的（IP都可以ping通）。

![img](l3.png "layer3 proxy")

L4常见代理有NAPT，[LVS](https://zh.wikipedia.org/wiki/Linux%E8%99%9A%E6%8B%9F%E6%9C%8D%E5%8A%A1%E5%99%A8)。4层重要信息有源目的端口和tcp状态，配合3层的源目的IP，就可以进行传输层代理，或者说负载均衡了。比如目的是ip1:port1，会转发到[ip2:port2, ip3:port3, &#x2026;]中的一个。在Kubernetes中的Service实际上就是一组Pod的负载均衡，service提供一个稳定的虚拟IP，指向一组Pod的IP（不稳定）和端口（Endpoints），每当有流量请求service的IP:port，按照轮询的方式将流量导入到某个Pod中。具体实现则是通过kube-proxy组件在宿主机节点上设置各种iptalbes规则（NAT，NAPT，端口转发等）实现（也可以使用LVS实现）。

![img](l4.png "layer4 proxy")

L7的Web则是我们最最常见的代理了，这一层是具体的应用层，持有的信息很多，可以读取消息中的内容来做决策，根据策略可以做很多事情，因此L7代理根据功能有很多别名，如负载均衡，缓存，[WAF](https://en.wikipedia.org/wiki/Web_application_firewall)，TLS [termination](https://en.wikipedia.org/wiki/TLS_termination_proxy)/[origination](https://istio.io/docs/tasks/traffic-management/egress/egress-tls-origination/)，gateway，正/反向代理，鉴权等。可以根据`Host`头和URL进行反向代理（工作在服务端），也可以对明文的HTTP加上TLS的正向代理（工作在客户端），还可以对请求URL路由到其他的服务，不一而足。

![img](l7.png "layer7 proxy")

好了，说了这么多，用那个好呢？

越底层的性能越好，也更通用，但是由于持有信息太少了，所以扩展性和易用性不如上层。

这个问题可以类比为存储中间件，我们有关系数据库，文档数据库，缓存，消息队列，倒排索引，对象存储这些选项。每个都各有千秋，关键在于我们业务的使用方式，只用一种存储通常是不够的。

比如在一个Kubernetes集群中，我们会给service配置一个外部的LoadBalancer（L3）或者NodePort（L4）；在我们的业务ingress gateway中，我们会使用L7代理进行各种策略，路由，TLS剥离，鉴权，熔断等；而Kubernetes集群本身网络则组合使用L2/3连通。

虽然网络将不同的机器连接在了一起使之能相互通信，这是分布式系统的基石之一。但是网络是不确定的，数据包可能丢失，可能无法连通，可能无序，也可能重发，超时，各种问题，所以在设计分布式系统的时候，要具有容错性，就算部分网络分区出错，整体功能仍然可用（[CAP](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)的P）。

说的比较浅，自己也仅仅是比较浅的认知，后续有机会和时间应该会找几个方面深入地去研究并写一些东西。涉及到的东西比较多，有不对的地方欢迎提出指正。

图片均来自网络。

## EOF

```yaml
summary: 浅谈网络代理和k8s的网络，包括L2/L3/L4/L7
weather: hot
license: cc-40-by
location: mars
background: ./proxy.png
tags: [network, k8s]
date: 2020-05-08T00:52:00+08:00
```
