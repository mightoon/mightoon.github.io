---
title: "[Equinix] Bare Metal as-a-service 体验之虚拟化部署"
date: 2021-9-21 11:15:37
description: 前面一篇文章概略地介绍了将我们的方案放在 Equinix 数据中心的原因，以及 Equinix Metal 的来龙去脉，本篇从实践角度记录在 Metal server 上构建 vSphere 集群的过程。
categories: Equinix
tags: [provisioning, virtualization]
---

[上一篇文章](https://mightoon.github.io/Equinix/Equinix-Part1-Introduction/)讲到 Equinix Metal 的来龙去脉，以及为什么我们选择把方案验证放在 Equinix 数据中心。本篇文中从实际部署的角度记录在 Metal server 上搭建 vSphere 集群的过程。这并不是一个 step by step 的说明，只是把那些有特点和需要注意的地方记录下来。

## Order，并生成基础 OS
我们的应用构建在由4台 `m3.large.x86` 节点组成的 vSphere 集群上。从名字的组成可以看出，Equinix Metal 遵循云上普遍的做法：large 是对其提供的 server 的规格的一个分类，'m'，以及其他诸如 'c'，'s' 等，用以区分对不同资源的侧重，'x86' 显然代表了 CPU 架构。除了 vSphere 集群，我们还 order 了额外两台 Linux server 作为对应用建立监控。所以一共在 Equinix Metal 上部署 4 台 ESXi 和 2 台 CentOS. CentOS 生成的速度非常快，从发起订单到初始化到灌入 OS image 再到 IP 可访问，仅一两分钟时间。ESXi 稍慢，也在数分钟内完成，可见优化了得。

## Server 的访问
### 关于 Public IP 地址 
在 provision server 的环节，可以选择是否需要 public IP. Equinix 会为每个 server 默认分配一个 public IP subnet，根据操作系统的不同，这个 subnet 是有大小区别的。

- 对于 Linux，分配的是 /31 的 subnet
- 对于 ESXi，会分配一个 /29 的 subnet

Public 网络以 Layer 3 的方式构建，因此网关会在 IP segment 中占据一个地址。Equinix 给出的 Public IP 是没有经过 NAT 的地址，实打实全球可达，这一点上，是比较慷概的。

需要注意的是:

1. Equinix 默认给的 Public 地址，是动态分配的，也就是说，server 重启之后，地址会变。对于 Linux 也许还好，然而 ESXi，往往受 vCenter 管理，地址变了会是一个问题。这种时候，需要申请 Elastic IP 地址，这是一种固定的地址，属于 Project 范围内的资源，可以绑定在同一个 Project 下除 ESXi 外（下面会讲原因）的任何 server 上，当然是需要付费的。

2. 上面提到，对于 ESXi，会默认分配 /29 的 subnet，看似不少，但是使用中我们也遇到问题。因为除去一个网络地址，一个广播地址，一个网关，一个 ESXi 的管理地址之外，实际上就只剩下4个可用地址。如果 VM 比较多又都需要 Public 地址，4个当然是不够的。一个解决办法是在 ESXi 内部署一台 route VM，如 vSRX 或 VyOS，专门来处理路由。

### 关于 Private IP 地址 
上一篇文章提到，Equinix Metal 中的资源往往有两个层面的属性，一是逻辑层面，一是物理位置层面。

Private 地址在逻辑层面属于 project 范围. Project 自带 /25 IP 地址池，在生成 Server 的时候，会从所属的 project 的地址池中取用，不够的话，也可以通过 elastic IP 来扩展
同时，private 地址在物理位置层面，属于 Facility 范围的资源，也就是说，同一个 project 中，在同一个 facility 下的 server，天生可以通过 private IP 通信。否则，需要借助 equinix fabric 创建更复杂的连接来满足 private 通信. 这在下一篇文章会提到。

### 安全访问
Equnix Metal 中 provision 的 server，在 ssh 方面，仅开放通过 key 登录，这点很好，但是没有看到类似 AWS security group 的设置，所以对来访没有专门的控制，对于 public IP 来说，还是比较危险的。我们的 vCenter 就因为过于频繁的访问被锁住了登录。

## vCenter 的部署
Equinix 不支持使用外部的 vCenter 管理 Metal Server 上的 ESXi，所以 vCenter 只能部署在 Equinix Metal 内部。有一个小地方需要注意，vCenter 安装好后，会通过 FQDN 访问，而这个 FQDN 是由 vCenter 的 hostname 和 Equinix 域名拼接起来的，需要在本地 DNS 中添加域名解析。如果没有更改 DNS 的权限的话，也可以将 vCenter 的 hostname 改成 localhost，这样可以强制使用 IP 访问，像我们这样的短期项目，也不是不可以接受。接下来就是在 vCenter 中创建集群，添加 Metal server，不再赘述。

## ESXi uplink 的问题
Equinix Metal 中的所有的 server，无论型号，都标配两个网卡（2个物理 port）. 对于 Linux 或 Windows 系统，两个 port 会被绑定为一个逻辑 port，到 public 网络和 private 网络都经过这个 port. 而对于 ESXi，情况有些不一样，ESXi 会创建一个 vSwitch，并且只增加一个物理 port 作为 uplink. 

{% img ESXi-uplink.png 600 ESXi_uplink %}

为什么不是两个？因为 Equinix Metal 环境中，与 metal server 对接的上级交换机统一配了LACP aggregation，而 ESXi 的vSwitch 有自己的聚合方式，是不支持 LACP 的。如果一定要把两个口用起来，需要在 vCenter 上创建 distributed vSwtich，把所管理的 ESXi 的端口都添加进去，然后在其上配置 LACP，就可以与上级交换机协商成功了。 

好了，我们的 vSphere 集群已经在 Equinix metal 环境中运行起来，下一篇文章将继续介绍，在集群中部署我们的一个自动化框架，需要在 Equinix Metal 中作的一些网络调整。



