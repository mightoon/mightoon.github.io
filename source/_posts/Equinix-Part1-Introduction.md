---
title: "[Equinix] 一个跨 co-location 和公有云的方案的落地"
date: 2021-10-16 09:07:37
description: 在一个 co-location 的方案验证中，我们将企业数据放在 Equinix IBX 数据中心，而消费数据的应用，我们选择放在了更加灵活的 Equinix Metal 和公有云上。 整个方案涉及到 Equinix Metal 通过 Equinix Fabric 与 Equinix co-location 数据中心的交互，以及与公有云的交互。这里准备通过四篇文章，概括 Equinix 的 bare metal 以及相关服务的体验，并以此窥探 Equinix 在数据中心和云方向的定位。  
categories: Equinix
tags: [BareMetal, co-location]
---

# 数据中心基础设施的选择

对 Equinix 最初的印象是全球 top 级的**数据中心托管平台**，通过自建或收购，在全球部署了两百多个数据中心，Equinix 将他们称作 `IBX`(International Business Exchange), 不少企业将自己的设备托管在 IBX, 以期利用其专业的 Infra 服务。之所以称为`最初印象`，是因为当时对 Equinix 的另一大优势业务尚未深入理解，实际上 Equinix 还提供全球最大的数据中心互联网络，通过 `Equnix Fabirc` 将企业，网络提供商，云服务商连接起来，在 Equinix 生态下，通过软件定义互联，方便地访问相互之间的服务。

我们此次设计的方案，类似企业级应用在混合云的部署，企业的数据存放在 co-location 托管的存储设备上，为企业应用提供数据底座，而具体的应用，部署在云下和云上。对于这种跨越 co-location 与公有云结构的支撑，正好是 Equinix 擅长的，这也是我们选择 Equinix 的原因。除此之外的另一个重要原因，是 Equinix 的 Bare Metal 业务。

Equinix Metal 现在成为 Equinix 几大主要业务之一，简单的讲，它让**物理机的部署具有类似公有云的体验**。实际上，Equinix Metal 是 Equinix 平台中比较新的一个业务，核心服务架构来自于对 [Packet](https://www.zdnet.com/article/equinix-to-buy-bare-metal-cloud-provider-packet/) 的收购，然后经过大半年的改造，于去年底在部分数据中心里面推出。

> The aim of Equinix Metal is to deploy physical infrastructure "at software speed" across the platform.

通过提供稳定的带外管理以及一套针对性开发的 API，Equinix Metal 将物理机的部署代码化，虚拟化，自动化。当然其实几大云厂商也提供 Bare Metal as-a-service，不过 Equinix Metal 在这个方面显得更专注一些，并且利用自家的 Equinix Fabric 的优势，通过灵活的软件定义网络，让客户能够快速将物理机连接到更远地方，诸如其他区域的 Equinix Metal 环境，或者客户自己的企业数据中心，亦或者公有云。说回到我们的项目，因为是方案验证，并非长期部署，使用 Equinix Metal 让我们不用折腾于把承担应用负载的 server 物理的运到 Equinix 数据中心，又能通过高质量的链路连接到我们部署在 Equinix IBX 的数据存储设备，同时，Equinix Metal 所在的数据中心，基本上都位于数据交换的主干节点，那也是离公有云最近的地方。这是 Equinix 连接公有云的优势，也完全匹配我们的要求。

在介绍具体部署细节之前，这里先梳理一下 Equinix Metal 中各种资源的组织方式。

# Equinix Metal 走马观花

## 资源组织层级
毕竟是收购来的部分，也可能是还在融合当中，Equinix Metal 有一个单独的 [Portal](https://metal.equinix.com/). 进入到 Portal 中首先需要把账号挂在某个 `organization` 下，这是统一管理 user 和 project 的地方，更重要的是，organization 是计费的主体。

每个 Metal 的设备，有两方面属性。
- 一是`虚拟属性`，即属于某个 project. `Project` 是 organziation 之下的一个逻辑层次，所有 metal 设备必须归属于某个 project
- 二是`物理属性`，具体来讲，就是设备属于哪个数据中心。在 Equinix Metal 的语境中，处于同一个地理位置的数据中心叫做 `Facility`，而临近的多个 Facility 组成一个 `Metro`. 在同一个 Metro 中，所有facility 天然地以高速链路相连接，而 Metro 也是有些资源的边界，比如 VLAN，比如 private IP，

## 资源类型
Equinix Metal 本质上依然是数据中心，其资源类型当然也逃不出计算、存储、网络等几大部分。

### 计算资源
即 bare metal server. 除了 server 的规格与位置等基础信息之外，与之相关联的，还有诸如订购方式，OS类型，SSH Key，Meta 数据，用户数据等。每个方面实际上是都是一个对象，可以通过 API 用代码指定或生成。

### 存储资源
Equinix Metal 提供各种类型的`本地存储`，以及外接的`企业级存储`设备。对于本地存储，跟所有的公有云体验类似，可以根据需要的性能选择盘的类型，大小。因为是针对 bare metal，还可以从配置 RAID 卡等底层设备。对于企业级存储，订购和配置都可以在 Portal 界面中完成，也可以通过代码，完成 provisioning 的过程。

### 网络资源
我们觉得，网络是 Equinix Metal 做得非常好的部分，毕竟 Equinix 靠提供高质量的互联起家，这部分基因也带到了 Equinix Metal. 所有的网络组件，全部实现 `software defined`，网络的搭建如同一个搭积木的过程，清晰而流畅。在 Equinix Metal 中，我们构建三层管理网络和二层数据网络，主要涉及到的组件包括：

- VLAN: 在 Equinix Metal 中的一个二层分区
- Metal Gateway: 用于二层网络对外通信
- Private IP: Metro 内部的 IP
- Backend Transfer：跨越 Metro 时需要的通信组件
- Elastic IP: 静态的可路由IP
- Dedicated Port: 用于将整个二层网络接入到 Equinix Fabric


下一篇文章将会利用这些组件，构建出我们需要的数据中心。
