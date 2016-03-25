Linux 中网络的配置
===

## 为什么配置静态IP很重要
linux服务器下配置静态的IP是一项**非常重要而且基本**的操作。

如果网络配置不成功，那么linux在网络上很多优秀的软件和性能都无从谈起。

所以，今天我们就来谈论一下如何在linux服务器上配置**静态IP**

## 配置方式
我们以**Centos 6.3**为例，其他的distros大同小异。

输入如下命令：

`vim /etc/sysconfig/network-scripts/ifcfg-eth0`

其中*eth0*指的是网卡号，你可以先使用`dmesg | grep 'eth'`去查看一下操作系统识别
到了那些网卡，然后根据识别到的网卡去选择ifcfg-xxx。

然后就是我们的配置的重点了：

---

```sh
DEVICE="eth0"
BOOTPROTO="none"
HWADDR="08:00:27:ED:FF:CF"
ONBOOT="yes"
NM_CONTROLLED="no"
TYPE="Ethernet"
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
NETWORK=192.168.1.0
BROADCAST=192.168.1.255
MTU=1500
```

---
下面来简单解释一下：

*``DEVICE``* 表示网卡设备名称，这个要和ifcfg-**xxx**一致才行。

*``BOOTPROTO``* 表示自动获得IP，这个我们当然不需要啦，填写**static**效果是一样的。

*``HWADDR``* 表示网卡的MAC地址，只有一块网卡的时候可以省略，但是**最好填上**。

*``ONBOOT``* 表示是否开机启动，是yes，否no。

*``NM_CONTROLLED``* 是否允许图形界面配置网络参数，当然不需要。

*``TYPE``* 网络类型。

*``IPADDR``* IP地址。

*``NETMASK``* 子网掩码。

*``GATEWAY``* 默认网关，注意，**如果有多块网卡，那么只能配置一个**。

*``NETWORK``* 网络。

*``BROADCAST``* 广播地址。

*``MTU``* 最大传输单元。

---

配置好之后，重启网络即可。

`service network restart`

然后你可以查看一下是否OK

`ifconfig eth0` 

> 实际上，最后三项可以不配置，``NETWORK``和``BROADCAST``可以由系统算出，
> 至于``MTU``看看自己网络的实际情况了。

## 联系我
>欢迎联系我，我的邮箱是im.longkai@gmail.com

>longkai，2013-07-03

### EOF
```json
{
  "tags": ["Campus", "Linux"],
  "render_option": 0,
  "date": "2014-01-13T23:33:00+08:00",
  "weather": "",
  "summary": "",
  "location": "Nanning",
  "background": "/assets/images/xida.jpg"
}
```
