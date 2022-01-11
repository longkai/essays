# 抛弃 TKE，搭建全新的 Kubernetes 技术栈

## 摘要

由于腾讯云CLB的大幅涨价，作为个人已无法负担，TKE已经失去了最大吸引力，于是起了动手自建k8s的念头，将自己这一年多的工作，学习的内容实践进去，均是业界最新的最佳实践。

本文即这一过程的流水账记录。

## 背景

我的个人网站和一些别的服务一直都部署在腾讯云的[容器服务TKE](https://cloud.tencent.com/product/tke)上。我觉得公有云的k8s最受欢迎的几个特点：

- LoadBalancer 类型的 Service
- dynamic persistence volume provisioner
- master 节点托管

master节点托管保证高可用，但对我这种单个worker node就够用的用户来说其实也无所谓。最关键其实是它作为Cloud Provider提供了云平台的IaaS/PaaS资源。

比如我要部署个web服务，需要动态的pv做存储，然后通过LoadBalancer类型的Service将服务暴露在公网上。声明一个pvc和svc然后apply一键搞定，就很舒服。

然而，最近腾讯云的CLB（LoadBalaner的实现）大幅涨价，好家伙，直接给我多了230块，要知道我原来每个月全部加起来也不到300，这让本不富裕的家庭雪上加霜。

既然用不起CLB了，似乎TKE就没有没啥吸引力了（动态pv数据迁移很麻烦，单节点其实也意义不大）。为何不自己搭建k8s，体验原生的最新的k8s，说不定还能自己去实现一些controler。

说干就干。

## 过程

### 初始化vm

选个至少2c2g的vm，我的是2c4g的香港节点，没那么多破事。然后系统选个顺手的，我选了最喜欢的Debian 10。

然后配置ssh key登陆，将内核更新到最新的稳定版，目前是5.10，这样做的目的是为了用上一些新特性，比如bbr和bpf等等，毕竟咱为了实验嘛。

内核更新参考这个朋友[文章](https://blog.dov.moe/posts/32240/)。

### containerd

接下来部署k8s的容器运行时，这里我们选择了containerd。k8s的下个版本1.24会移出docker运行时的支持，喜大普奔，此时不换，更待何时？

另外说到docker就来气，daemon的cs架构，占用资源高还要root权限，有了k8s很多功能也用不上，唯一的作用就是build镜像了，但是这样的话podman不香吗？另外，dockhub拉取镜像限频这骚操作也是恶心了人。

其实安装docker的时候也会同时containerd，这些都是docker的贡献。我们在k8s部署容易时，他的架构是这样的：

![]()

可以看到中间多了两个环节，最终起作用的还是containerd，删除这部分代码会减少维护成本并提升稳定性。

另外，在containerd中我们可以很容易的替换镜像的源：

todo：

containerd的[安装](https://docs.docker.com/engine/install/debian/)和[配置](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd)。

### kubeadm

搞定运行时后，我们就可以利用kubeadm这个官方提供的一个生产级别的工具来搭建集群。

安装的[步骤](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)很详细。

kubeadm提供了很多集群选项，特性的开关，为了方便以后每次使用，我们用一个yaml的配置文件来记录：

```yaml
# kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
# localAPIEndpoint:
#   advertiseAddress: k8s.xiaolongtongxue.com
#   bindPort: 6443 # better with 443, which needs a lb
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  # name: node-name
  taints: []
skipPhases:
- addon/kube-proxy
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  certSANs:
  - <your ip>
  - k8s.xiaolongtongxue.com
  timeoutForControlPlane: 4m0s
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
kubernetesVersion: 1.23.1
scheduler: {}
```

只需要`kubeadm init --config=kubeadm.yaml`即可一键部署。

值得注意的是，这里我们已经在yaml中删除了的taint，可以往master上调度Pod。

另外，我们把默认的`kube-proxy`组件给禁用掉了，why？

因为接下来我们部署网络插件cilium不需要它。

### 网络插件 cilium

此刻整个集群还是unready的，因为kubedns没起来，它需要一个网络插件。以前通常使用基于iptables或lvs插件，今天我们来体验一个基于eBPF技术的网络插件，[cilium](https://cilium.io)。

具体技术细节我其实不懂，但是从使用体验上来看，真的挺好，没有iptables那么多规则链，提供了crd的cloud native方式来控制网络，并且还有sidecar-less的service mesh，可观测性，能够在页面上看到网络的流向，并且，节点上可以直接访问Pod的ip，真神器也。

具体的安装利用helm还是蛮简单的：

```sh
$ helm install cilium cilium/cilium --version 1.11.0 \
    --namespace kube-system \
    --set kubeProxyReplacement=strict \
    --set k8sServiceHost=k8s.xiaolongtongxue.com \
    --set k8sServicePort=6443 \
    --set operator.replicas=1 \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true
```

没过多久集群就会ready了。

网络这块后面值得花时间研究，先往下走。

### Istio

我其实一直在用Istio作为ingress gateway，这小破网站service mesh其实用不上，反而是下面这些特性让我很喜欢：

- envoy的xDS可玩性很高
- gateway tls termination
- 基于 tls/websocket 的服务路由
- 自动tls证书管理（配合cert-manager）
- 在 sidecar 中进行 grpc-json transcode（省略grpc-gateway server）

之前用的1.9，现在1.12了，简单看了下，基于CRD的wasm plugin还是停看好了，学了rust倒是可以玩一下hhh。

不推荐istioctl的部署，比较麻烦，也不适合管理，1.12终于又提供基于helm的安装。

> 这里插一句，istio的安装方式有istioctl或者operator，本质上都是利用helm来配置和渲染yaml manifest。

helm分了好几个chart

- base（crd）
- istiod（控制面）
- istio-ingress（ingress gateway）

要注意istiod和ingress不再同属一个命名空间了（istio-system），在看它的官方的文档中，基本上还是按照都在一个ns来说明的，这个坑要注意下。这里不是bug，而是有意而为之，为了更好的控制ingress。

istio的安装的values需要注意修改一下默认值，考虑到我们是2c4g的单机，另外没有LoadBalancer（CLB）可以用了：

```yaml
# istiod.yaml
pilot:
  resources:
    requests:
      cpu: 10m
      memory: 10Mi
```

```yaml
# ingress.yaml
global:
  proxy:
    # Resources for the sidecar.
    resources:
      requests:
        cpu: 10m
        memory: 10Mi
      limits:
        cpu: 2000m
        memory: 1024Mi

service:
  # Type of service. Set to "None" to disable the service entirely
  type: ClusterIP
resources:
  requests:
    cpu: 10m
    memory: 10Mi
  limits:
    cpu: 2000m
    memory: 1024Mi
```

接下来自动管理tls证书，毕竟现在是HTTPS的时代，没个证书很不方便。

### cert-manager

Let's Encrypt 是免费的证书CA机构，提供了自动化的方式来签发和更新tls证书。使用体验非常好，还支持通配符，唯一不好的就是有效时间三个月。

从安全角度来说这也没啥不好，但从运维角度来说有点麻烦，需要设置crontab以及停掉http server来进行证书更新，我之前都是这么干的。

现在有了更自动化，更cloud native的做法。简单来说通过Let's Encrypt调用域名服务商的api来验证这域名的所有权，然后签发证书。

[cert-manager](https://cert-manager.io)这个利用k8s的crd，抽象了证书管理的过程，apply几个yaml，以后再也不用考虑证书问题了。

- Issuer 证书签发者
- Certificate 证书声明

```yaml
# 声明一个证书的签发者
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: longkaify@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # ACME DNS-01 provider configurations
    solvers:
    - dns01:
        # Here we define a list of DNS-01 providers that can solve DNS challenges
        cloudflare:
          email: my-cloudflare-acc@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

```yaml
# 这里声明了一个证书，包括域名和签发者引用，以及最后证书签发后存在哪个secret里
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ingress-cert
spec:
  secretName: ingress-cert
  issuerRef:
    name: letsencrypt-prod
    #kind: ClusterIssuer
  commonName: "*.xiaolongtongxue.com"
  dnsNames:
  - xiaolongtongxue.com
  - "*.xiaolongtongxue.com"
```

不出意外的话，certificate状态会很快ready，然后`ingress-cert`的setret资源里就包含`tls.key`和`tls.crt`了。

证书搞定了，那么最后怎么更新到ingress上去呢？我可不想像nginx那样手动reload一下，有没有一种自动化的方式呢？

有，Envoy（也就是istio）的Secret discovery service (SDS) 就是专门来干这事的，它可以通过监听文件变化，或者利用gRPC的双向stream等方式，来感知证书的变化，从而实现证书的无缝更新。这体验没谁了。

具体到istio的gateway配置：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingress # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "xiaolongtongxue.com"
    - "*.xiaolongtongxue.com"
    tls:
      httpsRedirect: true # sends 301 redirect for http requests
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: ingress-cert # This should match the Certificate secretName
    hosts:
    - "xiaolongtongxue.com" # This should match a DNS name in the Certificate
    - "*.xiaolongtongxue.com"
```

这里特别注意，证书生成的secret必须和ingress在同一个ns。

最后，cert-manager的安装也很简单，helm一把搞定：

```sh
helm install cert-manager bitnami/cert-manager --set installCRDs=true
```

网络搞定，下一个就是存储了。

### 存储

前面提到过，基于云服务厂商的动态pv多半是云硬盘，其实对于我这样的单节点小破站没用，还增加了存储的迁移负担。

最好的方式其实nfs文件存储，各个pod都能读写，性能倒其实不重要了。

腾讯云倒也提供了nfs的服务，很便宜，但是问题也蛮多：

- 有vpc的限制，同地域不同区死活访问不了，也是醉了
- 迁移麻烦
- 无法预览
- 缺少备份
- 无API

API那个倒没试过，但是想了想现在走都是对象存储，nfs应该是没有的。有没有一种一种存储可以同时支持上述的特性呢？

是时候祭出大杀器了，Dropbox！

Dropbox免费的空间够用了，三个终端实时的进行文件增量更新，也不用担心迁移负担，删掉节点，新建节点，文件都还在，我们可以在网页上预览文件，甚至调用它的sdk干点别的上传下载。

所以用法也很简单，节点上搭建nfs服务器，把本地dropbox的文件目录挂载到nfs里，然后作为k8s的pv，最后在不同的服务里用pvc引用，简直不能再棒！

Pod中的文件更新了，会反应到节点的Dropbox目录，最后同步到dropbox到服务器，然后我mac上的共享目录也会更新。反之亦然，我也能够在mac修改文件，反应到pod里。

这样的存储特性不能再棒，兼具易用性，高可用，扩展性，还免费。

网络存储都具备，只欠应用了。

### 应用部署 all in one chart

目前有4个服务部署，分别是：

- www网站
- webdav文件服务
- 下载服务
- ss

原来用的都是raw的yaml逐个apply，最近项目里大量使用了helm，于是就顺手将所有的服务做成了chart，真正的实现了一键部署管理。

另外，另外前面提到的泡在k8s上各种服务，基本都是helm安装，这里等helm处理好子chart的命名空间后可以更进一步把所有依赖都放在一个all in one chart里。

最后将istio的virtual service路由到各个服务中，整个网站就起来了。但是，外网还是访问不了我们的，因为ingress gateway缺失loadbalancer，也就是CLB，我们一开始提到的问题，放到最后来解决。

### Load Blancer

k8s服务暴露到集群外一般有三种方式：

- host network（或者host port）
- nodeport
- loadbalancer

第三种没有，第一种配置太复杂（涉及到sidecar和root一大堆配置），难以维护放弃，第二种不是标准80/443端口，需要额外load balancer转发过来（其实这也是clb的实现方式）

虽然三种都不可用，但是我们可以将三者串起来就可以实现我们自己的loadbalaner了。具体来说：

- envoy在节点上进行l4代理，端口转发到istio ingress中
- 将envoy通过deploy部署到k8s中，host port直接监听80/443端口，并且能够识别ingress的dns，这样更易于维护
- 基于上一点，nodeport不需要了，直接使用ClusterIP，暴露更少的端口

非常好，唯一的问题，就是获取源IP地址很麻烦，但是也不是没有办法，因为现在业务不需要考虑源IP地址，所以暂时就不提了，后面有需要再说。

envoy tls的proxy配置如下：

```yaml
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
    filter_chains:
    - filters:
      - name: envoy.filters.network.tcp_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
          stat_prefix: destination
          cluster: istio-ingress-443
```

这里是静态的LDS(listner discovery service)，省略了EDS(endpoint discovery service)配置，其实envoy的配置都是proto形式的grpc，初看起来是有些看不懂，但是搞清楚里面的结构后，还是比较清晰的，顺着rpc接口和proto描述，xDS组合起来能实现一些比较复杂的功能。

## 总结

这是一次充满挑战和愉快的部署经历，不是简单跑起来，而是把工作和学习的内容结合了当前云原生领域的一些新特性和最佳实践，从网络，存储，应用，配置等方面，自动化串起来，沉淀一些方法和经验。

更近一步，对于这里面感兴趣地方可以去研究一下设计和实现，以后自己在项目中也能组合借鉴起来。
