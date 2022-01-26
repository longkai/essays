# 用k8s来解决问题而不仅仅是部署服务

## 摘要

由于近期腾讯云[CLB](https://cloud.tencent.com/product/clb)费用大涨，作为个人已无法负担，于是[TKE](https://cloud.tencent.com/product/tke)失去了最大吸引力。考虑节约成本和技术实践，起了自建k8s的念头，遂将自己这一年来的工作，学习内容通过个人网站落地，做一个最佳实践吧。

本文即对这一过程的一些记录，方便后续查漏补缺。按照用什么东西解决什么问题的方式，主要讨论了**自建k8s的最佳实践，网关的选择，怎样做tls证书的自动化管理，应用的一键部署，没有cbs用dropbox做pv存储方案以及没有clb的情况下利用xDS和API-server将集群流量暴露出去**等解决方案。最终回归主题：把k8s作为解决问题的一个工具，而不仅仅是用来部署容器服务。

过程和原理我会根据感兴趣的程度后续写一些较深入的文章。

## 背景

我的[个人网站](https://xiaolongtongxue.com)维护了有4、5年了，从最开始的静态页面到二进制部署再利用docker以及目前基于TKE，也算是过来人了吧hhh。我觉得公有云的k8s最受欢迎的几个特点：

- LoadBalancer 类型的 Service
- dynamic persistence volume provisioner
- master 节点托管

master节点托管保证了控制面的高可用，但对我这种单个worker node就够用的钉子户来说其实也无所谓。最关键其实是它作为[Cloud Provider](https://kubernetes.io/zh/docs/tasks/administer-cluster/running-cloud-controller/)提供了云平台的IaaS/PaaS资源。比如我要部署个web服务，需要动态的pv做存储（云硬盘），然后通过LoadBalancer类型的Service（负载均衡器）将服务暴露在公网上。声明一个pvc和svc然后apply一键搞定，就很舒服。

好景不长，最近腾讯云的CLB（LoadBalaner的实现）大幅涨价，好家伙，直接给我多了230块，要知道原来每个月费用全部加起来也不到300，这让本不富裕的家庭雪上加霜。

既然用不起CLB了，似乎TKE就没有没啥吸引力了（动态pv数据迁移很麻烦，单节点其实也意义不大）。为何不自己搭建k8s，体验原生的最新的k8s，说不定还能自己根据需要去实现一些controller。

说干就干，先明确目标。

## 目标

总体目标其实可以归为一句话：**云原生，自动化，一键部署所有服务**。

直白点说就是**用k8s来解决问题而不仅仅是部署服务**，让这一切都自动化的同时自己尽可能地减少ad-hoc的东西来搞事情。

比如，

- tls证书如何签发？证书又应该如何下发到网关服务？整个过程服务要保证高可用和完全自动化？
- 没有了load balancer，我们要怎么样把服务队外暴露出去？这过程能不能啥也不配置一键搞定？
- 存储如何解决？数据迁移的成本能否降到零？

这些问题其实一直伴随着网站的整体架构的变迁，比如说证书的处理，经历了在证书机构下载证书传到server上，修改nginx配置，再reload；后面利用crontab去停服更新[Let's Encrypt](https://letsencrypt.org)的证书的过程。总体上说不是那么自动化，没有用到k8s的精髓。

下面从问题出发，*用x来解决y*的模式来简单的记录本次迁移的方案。先从部署k8s开始。

## 如何部署k8s？

方案如下：

1. 部署容器运行时[containerd](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd)
2. [kubeadm](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)部署master节点
3. 最后网络插件使用基于[eBPF](https://ebpf.io)的[cilium](https://cilium.io)

docker作为k8s的容器运行时环境dockershim将会在下一个k8s版本1.24被[移除]((https://kubernetes.io/blog/2021/11/12/are-you-ready-for-dockershim-removal/))，替代方案之一就是containerd，本质上docker绕了两层最终用的也是这个运行时，此时不换更待何时？另外kubeadm部署的时候最好采用yaml的配置文件，方便下次再用，记得别安装kube-proxy组件，因为cilium不需要。

> 这里有些技术细节可以留着后面细说，总之这三个搭配现阶段不会错的。

## 网关服务用什么？

其实一直在用Istio作为南北的ingress gateway（nginx在云原生领域有一些老旧了），这小破网站service mesh其实用不上，反而是下面这些特性让我很喜欢：

- envoy的[xDS](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)可玩性很高
- gateway tls termination
- 基于 tls/websocket 的服务路由
- 自动tls证书管理（配合cert-manager）
- 在 sidecar 中进行 grpc-json transcode（抛弃grpc-gateway server）

不推荐istioctl的部署，比较麻烦，也不适合管理，1.12终于又提供基于helm的安装。

> 这里插一句，istio的安装方式有istioctl或者operator，本质上都是利用helm来配置和渲染yaml manifest。

istio的安装的values需要注意修改一下默认值，考虑到我们是2c4g的单机，另外没有LoadBalancer（CLB）可以用了。

接下来自动管理tls证书，毕竟HTTPS的年代，没个证书很不方便。

## 如何自动管理tls证书？

这个问题前面有提过，这里直接说方案。[cert-manager](https://cert-manager.io)，这是一个operator，利用k8s的crd，抽象了证书管理的过程，这是用k8s来解决问题的一个极好栗子。

本质上 Envoy（也就是istio数据面）的[Secret discovery service (SDS)](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret)就是来专门来干这事的，它可以通过监听文件变化，或者利用gRPC的双向stream等方式，来感知证书的变化，配合k8s的secret volume从而实现证书的无缝更新。这也体验没谁了，丝滑！

网络搞定，下一个就是存储了。

### 没有cbs怎么办？迁移成本？

前面提到过，基于云服务厂商的动态pv多半是云硬盘，其实对于我这样的单节点小破站没用，还增加了存储的迁移负担。试想每次创建一个服务，都给我创建一个新云硬盘，每次都要导入数据进这硬盘就很麻烦，数据同步更蛋疼。

最好的方式其实nfs文件存储，各个pod都能读写，性能倒其实不重要了。

腾讯云倒也提供了nfs的服务，很便宜，但是问题也蛮多：

- 有vpc的限制，同地域不同区死活访问不了，也是醉了
- 迁移麻烦
- 无法预览
- 缺少备份
- 无API

API那个倒没试过，但是想了想现在走都是对象存储，nfs应该是没有的。有没有一种存储可以同时支持上述的特性呢？

Dropbox举手！

免费的空间够用了，三个终端实时的进行文件增量更新，也不用担心迁移负担，删掉节点，新建节点，文件都还在，我们可以在网页上预览文件，甚至调用它的sdk干点别的上传下载。

所以用法也很简单，节点上搭建nfs服务器，把本地dropbox的文件目录挂载到nfs里，然后作为k8s的pv，最后在不同的服务里用pvc引用，简直不能再棒！

Pod中的文件更新了，会反应到节点的Dropbox目录，最后同步到dropbox到服务器，然后我mac上的共享目录也会更新。反之亦然，我也能够在mac修改文件，反应到pod里。

这样的存储特性不能再棒，兼具易用性，高可用，扩展性，还免费。

网络存储都具备，只欠应用了。

### 一键部署所有的应用？

目前有4个服务，分别是：

- www网站
- webdav文件服务
- 下载服务
- ss

用的都是raw的yaml逐个apply，就像临时脚本一样不好管理。最近项目里大量使用了helm，于是就顺手将所有的服务做成了chart（dependency），真正的实现了一键部署管理。

最后将istio的virtual service路由到各个服务中，整个网站就起来了。但是，外网还是访问不了我们的，因为ingress gateway缺失loadbalancer，也就是CLB，我们一开始提到的问题，放到最后来解决。

### 怎么把服务暴露到集群外？

k8s服务暴露到集群外一般有三种方式：

- host network（或者host port）
- nodeport
- loadbalancer

第一种配置太复杂（涉及到sidecar和root一大堆配置），影响调度，难以维护放弃，第二种不是标准80/443端口，需要额外的外部load balancer转发过来（其实这也是clb的实现方式），所以我们选择第三种使用外部的自建loadbalancer。

利用Envoy的[Listener discovery service (LDS)](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/lds)和[Endpoint discovery service (EDS)](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery-types-eds)方案配合k8s本身的服务发现和 API server 的能力，lb其实还是比较容易实现的。这也是envoy对比nginx的优势，API first。

非常好，唯一的问题，就是获取源IP地址很麻烦，但是也不是没有办法，因为现在业务不需要考虑源IP地址，所以暂时就不提了，后面有需要再说。

走到这一步，网站已经可以对外服务了。

> 整个过程非常有挑战和有趣，这里值得一波深入的讨论。

## 总结

这是一次充满挑战和愉快的经历，把最近一时期的工作和学习的内容结合了当前云原生领域的一些新特性和最佳实践串起来：关键是要把作为k8s解决问题的工具，而不是简单的容器服务。

更近一步，对于这里面感兴趣地方可以去深入研究一下设计和实现，以后在项目中也能借鉴起来。

## EOF

```yaml
summary: 由于近期腾讯云CLB费用大涨，遂起了自建k8s的想法。这篇文章主要讨论了自建k8s的最佳实践，网关的选择，怎样做tls证书的自动化管理，应用的一键部署，没有cbs用dropbox做pv存储方案以及没有clb的情况下利用xDS和API-server将集群流量暴露出去等解决方案。
weather: cold&rainy
license: cc-40-by
location: Guilin
background: ./photo-1439405326854-014607f694d7.jpeg
tags: [k8s, envoy, xds, devops]
date: 2022-01-26T17:30:00+08:00
```
