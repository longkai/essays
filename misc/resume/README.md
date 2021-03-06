
# 龙凯

-   13168001399
-   广东 - 深圳 - 南山
-   im.longkai@gmail.com
-   <https://www.xiaolongtongxue.com>
-   <https://m.douban.com/people/60187253>
-   意向岗位：Go 后台开发，Kubernetes 容器平台相关开发

## 技术能力

-   熟悉 Go/Java，了解并使用过 C/JavaScript/Swift 等语言
-   熟悉 Go 环境下的 REST/gRPC 服务端开发
-   了解 MySQL，Redis 等存储中间件
-   熟悉 Android 开发，了解 iOS/Web/小程序 的开发
-   熟悉 Kubernetes/Docker 平台相关的计算和网络，熟悉 Istio 服务网格
-   熟练使用 Linux 进行日常工作开发，喜好折腾各种效率和网络工具，如 Emacs/Vim/curl/ss/tcpdump/Wireshark
-   熟悉常用的数据结构和算法，并发编程，了解分布式系统设计，对 TCP/IP 协议栈有较深的理解，如 TLS，HTTP，HTTP/2，WebSocket

## 项目经历

### 深圳瓶子科技（腾讯全资子公司） - 客户端/后台 开发工程师（2017-5 至今）

-   腾讯云长沙市超脑项目 / 健康长沙门户 （19年4月至今）
    -   技术负责人
    -   设计并实现了一个 Non-Blocking 的获取第三方 API token 服务，安全高效地解决了不同版本，环境的应用间并发获取 token 资源的问题
    -   设计并实现了一个使用 DSL 描述 HTTP 接口调用与适配的库，包含字段映射，类型转换，格式化，请求依赖等，大大简化了适配不同供应商接口代码的编写
    -   将业务 API server 的接口重构为 Google 的最佳实践，统一了接近70%的接口风格，增加版本控制，更稳定方便和客户端对接
    -   在项目中紧跟标准，如参考 [RFC6749](https://tools.ietf.org/html/rfc6749) 和 [RFC7807](https://tools.ietf.org/html/rfc7807)分别实现了OAuth2授权，无状态的JWT&JWK认证，以及统一友好的接口错误，易于接入和快速定位问题
    -   引入 Protocol Buffers/gRPC 通信协议，并在 Envoy sidecar 上实现了 gRPC-JSON 转码，同时支持 REST/JSON 调用，减少了前后端对接时间，提升了后台30%的开发效率
    -   主导项目从单个大应用，转换为各个微服务的过程；搭建 Kubernetes 集群，将所有的服务迁移到 k8s 上，大大提高了项目的敏捷度并扩宽了团队的技术栈
    -   使用 Istio 服务网格进行流量管理，路由，监控，发布新版时线上服务 zero down-time，并且多个分支进行A/B测试互不影响，开发与测试更容易
    -   自动化单元测试，私有Docker镜像构建，自动部署等工作流程并与工蜂和企业微信消息连接，提升了开发体验与效率
-   求知鸟付费课程项目 (18年3月至19年4月）
    -   学习并使用 SpringBoot 开发微信公众平台相关的业务
    -   负责运营后台和推荐系统的开发，提升了10%用户的活跃度
    -   搭建ELK栈，收集各个系统的日志，并通过规则监控，方便定位问题
-   活点地图项目 / Android&iOS（17年7月至18年2月）
    -   负责核心的好友地图交互，即时聊天里表情动画的开发

### 深圳市时时医科技有限公司 - Android 开发工程师 (2015-6 至 2016-2)

-   神经脊柱医患 Android
    -   技术负责人
    -   负责即时通信模块，方便医生与患者沟通

### 深圳市爱美购科技有限公司 - Android 开发工程师 (2014-6 至 2015-6)

-   么么嗖海淘 Android
    -   负责首页，商品详情，购物车，订单，微信／支付宝支付，物流查询，自定义控件等模块；其中前半年里只有自己一个人开发，从零到上线
    -   在 Android 端实现了自定义模版引擎，将服务端下发的布局描述文件渲染成原生界面，使得无需提交新版，便能快速地上线一些运营活动

## 教育

广西大学 - 计算机科学与技术 - 本科 (2010 至 2014)

英语六级，能够熟练地阅读和撰写英文文档

## EOF

```yaml
hide: true
license: all-rights-resered
location: 22, 114
tags: [Career]
title: Resume
weather: fine
date: 2017-04-17T12:17:00+08:00
```
