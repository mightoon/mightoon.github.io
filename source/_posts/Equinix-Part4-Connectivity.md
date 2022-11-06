---
title: "[Equinix] 从 Equinix Metal 到 Equinix IBX 数据中心"
date: 2021-9-29 13:18:41
description: 本系列的前面几篇文章，主要围绕如何在 Equinix Metal 数据中心内部部署我们的应用。从这篇文章开始，范围将延申到 Equinix Metal 之外，讨论如何从 Equinix Metal 的应用上访问我们托管在 Equinix IBX 数据中心的存储设备上的数据。
categories: Equinix
tags: [Network, Fabric]
---

# 从 Equinix Metal 向外访问
Equinix Metal 数据中心向外访问的场景，大概有这几种情况：
- 某个区域的 Equinix Metal 数据中心访问其他区域的 Equinix Metal 数据中心
- 从 Equinix Metal 数据中心访问 Equinix IBX 数据中心
- 从 Equinix Metal 数据中心访问 Equinix 生态中的服务，或者公有云

对了，这里列举的访问场景，都是基于 private 的相对低延时的访问。基于公共 internet 的访问不在讨论之中。
针对第一种情况，其实在前面的文章中有提到过，如果两个 Equinix Metal 数据中心属于同一个 Metro，那么天生就具有高速连接，直接免费访问。如果两个 Equinix Metal 数据中心位于不同的 Metro，也很简单，打开 Equinix Metal 数据中心的 `backend transfer`，即可实现跨 metro 访问，只是需要额外的费用。

第三种情况，因为 Equinix 生态中的服务一般都接入了 Equinix Fabric，所以需要将 Equinix Metal 也接入 Equinix Fabric，就可以实现互访。同样的原理，访问公有云亦如此。

第二种情况就就是我们的场景，需要从 Equinix Metal 数据中心访问我们托管在 Equinix IBX 数据中心的存储设备，比较方便的方式还是通过 **Equinix Fabric**，这是一个全程 software definded 的体验。但如果对延时要求很高，还可以向 Equinix 申请 **Cross Connect**，需要一定的实施时间。

# 跨数据中心的连接
Equinix Metal 与 Equinix fabric 配合的过程

share port and dedicated port

QinQ技术
之所以能 software defined，

创建过程

不支持 LAG 的问题


