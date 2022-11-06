---
title: "[Equinix] Bare Metal as-a-service 体验之虚拟化部署"
date: 2021-9-21 11:15:37
description: 前面一篇文章概略地介绍了将我们的方案放在 Equinix 数据中心的原因，以及 Equinix Metal 的来龙去脉，本篇从实践角度记录在 metal server 上构建 VMware vSphere 集群的过程。
categories: Equinix
tags: [Provisioning, Virtualization]
---

[上一篇文章](https://mightoon.github.io/Equinix/Equinix-Part1-Introduction/)简单梳理了 Equinix Metal 的来龙去脉，以及为什么我们选择把本次方案的验证放在 Equinix 数据中心。本篇文章从实际部署的角度记录在 metal server 上搭建 VMware vSphere 集群的过程。这并不是一个 step-by-step 的说明，只是把那些具有特别之处和需要注意的地方记录下来。

## 从下订单到操作系统就绪
我们的应用构建在由4台 **m3.large.x86** 节点组成的 vSphere 集群上。从名字的组成可以看出，Equinix Metal 遵循云上普遍的做法：`large` 是对其提供的 server 的规格的一个分类，`'m'`，以及其他诸如 `'c'`，`'s'` 等，用以区分对不同资源的侧重，`'x86'` 显然代表了 CPU 架构。除了 vSphere 集群，我们还预定了额外两台 Linux server 用于建立应用的监控。所以一共在 Equinix Metal 上需要部署 4 台 ESXi 和 2 台 CentOS.

CentOS 生成的速度非常快，从发起订单到初始化然后灌入 OS image 再到 IP 可访问，全程只需一两分钟时间。ESXi 稍慢，不过也在数分钟内完成，可见优化了得。

## Server 的访问
### 关于 Public IP 地址 
在 server provisioning 环节，有机会选择是否需要 public IP . Equinix 会为每个 server 默认分配一个 public IP subnet，根据操作系统的不同，这个 subnet 是有大小区别的:

- 对于 Linux，会分配一个 /31 的 subnet
- 对于 ESXi，会分配一个 /29 的 subnet

Public 网络默认以 Layer 3 的方式构建，因此网关会在 IP subnet 中占据一个地址。Equinix 给出的 Public IP 是没有经过 NAT 的地址，实打实全球可达，这一点上，是比较厚道的。

需要注意的是:

1. Equinix 默认给的 Public 地址，为动态分配，也就是说，server 重启之后，IP 地址可能会变。对于 Linux 也许还好，然而 ESXi，通常需要通过 vCenter 管理，地址变了会是一个问题。这种时候，往往需要向 Equinix 申请 Elastic IP 地址，这是一种固定的地址，属于 project 范围内的资源，可以绑定在同一个 project 下任何 server 上，当然，是需要付费的。

2. 上面提到，对于 ESXi，默认分配 /29 的 public IP subnet，看似不少，但是使用中我们也遇到问题。因为除去一个网络地址，一个广播地址，一个网关，和一个 ESXi 的管理地址之外，实际上就只剩下4个可用地址。如果 VM 比较多又都需要 public 地址，那么4个当然是不够的。一个解决办法是在 ESXi 内部署一台 route VM，如 vSRX 或 VyOS，专门来处理路由。

### 关于 Private IP 地址 
上一篇文章提到，Equinix Metal 中的资源往往有两个层面的属性，一是**逻辑层面**，一是**物理位置层面**。

- Private IP 在逻辑层面属于 project 范围。 创建 project 的时候，会附带一个 /25 IP 地址池，新生成的 server，会从所属的 project 的地址池中取用地址，不够的话，也可以通过 elastic IP 来扩展。
- Private IP 在物理位置层面，则属于 facility 范围的资源，也就是说，同一个 project 中，在同一个 facility 下的所有 server，天生可以通过 private IP 通信。否则，就需要借助 `Equinix Fabric` 创建更复杂的连接来满足 private 通信。在后面第四篇文章中会详细讨论。

### 安全访问
Equnix Metal 中 provision 的 server，在 ssh 方面，仅开放通过 key 登录，这点很好，但没有看到类似 AWS security group 的设置，所以针对来访并没有专门的控制，这样对于 public IP 而言，还是比较危险的。我们的 vCenter 就因为遭遇 flood 攻击被锁住了登录。

## vCenter 的部署
Equinix 不支持使用外部的 vCenter 管理 Metal Server 上的 ESXi，所以 **vCenter 只能部署在 Equinix Metal 内部**。有一个小地方需要注意，vCenter 安装好后，会通过 FQDN 访问，而这个 FQDN 是由 vCenter 的 hostname 和 Equinix 域名拼接而成，需要在本地 DNS 中添加域名解析。如果没有更改 DNS 的权限的话，也可以将 vCenter 的 hostname 改成 localhost，这样可以强制使用 IP 访问，像我们这样的短期项目，也不是不可以接受。接下来就是在 vCenter 中创建集群，添加 ESXi server，不再赘述。

## ESXi Uplink 的问题
Equinix Metal 中的所有的 server，无论型号，都标配两个网卡（一共2个物理网络端口）。对于 Linux 或 Windows 系统，两个端口会被聚合为一个逻辑 port，去到 public 网络和 private 网络都经过这个逻辑 port. 而对于 ESXi，情况有些不一样，ESXi 默认的通信模式是通过 vSwitch，只会有一个物理端口加入到 vSwitch 中作为 uplink，而且无法添加第二个物理端口。 

{% asset_img ESXi-uplink.png 680 ESXi-uplink %}  

为什么不能是两个？这个问题在一开始困扰了我们一段时间，后来了解到 Equinix Metal 环境中，与 metal server 对接的上级交换机统一配置了LACP aggregation，而 ESXi 的vSwitch 有自己的聚合方式，是不支持 LACP 的。我们作为 Equinix Metal 的客户，它的上级交换机对我们是透明的，也无法作任何更改，那么如果一定要把 metal server 的两个口用都起来，有一个替代的办法，就是抛弃 vSwitch，改为在 vCenter 上创建 distributed switch，然后在其 uplink 上配置 LACP，就可以与上级交换机成功协商了。 

好了，我们的 vSphere 集群已经在 Equinix Metal 环境中运行起来。下一篇文章将继续介绍，在集群中部署我们的一个自动化框架，需要在 Equinix Metal 中作的一些网络调整。



