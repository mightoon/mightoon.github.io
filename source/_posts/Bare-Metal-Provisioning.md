---
title: 有关 Bare Metal Provisioning 的一些想法
date: 2017/8/19 18:07:25
description: 最近终结了一个项目，是关于数据中心种裸机的自动部署方案。 本篇文章写一点对 Bare Metal Provisioning 的想法。
categories: Automation
tags: [DevOps, BareMetal]
photos: https://www.codeclouds.com/wp-content/uploads/2019/01/38.png
---


Bare Metal Provisioning 的实现，通常依赖于一个相当成熟的技术： **IPMI**
  
服务器往往需要远程管理，有 OS 的时候可以通过 OS. 那么，在还没有 OS ，或者 OS 无法响应的时候，怎么远程控制服务器?  

一种普遍的方式是通过`IPMI`  
  > IPMI 定义管理员如何监测系统硬件，传感器，控制系统组件，日志等  
  > 即使系统崩溃也可以获得机器状态、重要系统日志，还能实现系统的重启、关机等控制

IPMI的实现需要一个专门的芯片，叫做 **BMC**. 主流厂商的服务器都有这个芯片，HP，Dell, Cisco 等等。BMC 不依赖于服务器的处理器、BIOS 或操作系统，有的是服务器上一个独立板卡，有的直接集成在主板上，而 IPMI 定义了 BMC 做事情的规范。远程访问即是用到了IPMI定义 `Serial Over LAN` 的特性, SOL 通过 IPMI截取数据，将本来发往串口的信息重定向到局域网端口，这就提供了远程查看 BOOT、OS 加载等功能

继续说一下我的粗浅理解，不一定全面：  
云，对资源的调度还是有层次之分的。上层，对服务的管理，对租户的管理，做得比较成熟了；底层，即 IaaS. 这一点 VMWare 做得比较好，面向的是装有 Hypervisor 的 Host，对于 OpenStack 而言，针对资源组的 provisioning. 而再往下，VMWare 很难下去了，OpenStack 有，但是一直做得不是很好。  
那么问题来了，如果没有先装好 Hypervisor 呢？所以需要Bare Metal Provisioning ，可以说是填补了Cloud Stack 比较薄弱的一环。IT 公司都看到了问题，如EMC，所以搞了个 RackHD。

<br/>

**[RackHD](https://github.com/RackHD)** 是 EMC 开源的，一个平台无关的堆栈，目的是要解决大规模环境下管理和组织协调服务器与网络，存储资源这个 VMware，Openstack 还没有解决好的问题。网络和存储先不多说，服务器这一步，是比较清楚的，就是 Bare Metal Provisioning. RackHD 想做到可以自动发现、描述、调配、编程各种服务器。  
开发方面，可使用 RackHD API 作为组织协调系统的一个组件，或者创建一个用户界面管理硬件服务而**无需考虑底层硬件是否就绪**，这就很厉害了，就是说有了它，通过 vRealize 或者 Openstack 直接调用RackHD，之后的数据中心就可以完全从物理硬件到服务，全部自动化起来了。





