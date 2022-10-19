---
title: Flow Vision 流量监视
date: 2017-7-21 14:01:33
description: CMCNE 发布 v14.2.1 版本，功能跟以前没有太大变化。注意到 Flow Vision 这一部分比较有意思，本篇文章即记录一些关于 Switch Flow Vision 的尝试。 
categories: Switch
tags: [FC, Flow]
---

# Brocade Flow Vision 
Flow Vision 是 Brocade 上的用来诊断 FC SAN 的工具，主要有三个功能
- 监测 traffic flow
- 复制 traffic flow 用以分析 bottleneck, bandwidth utilization, slow drain 之类
- 主动 generate flow 用以检测 SAN 的各种 connectivity

# 关于 Flow
`Flow` 在这里指的是具有相同特征的一系列 frame，包含两方面信息：一是 port，比如他们都要通过同一个 ingress port 或 egress port，二是frame内容信息，比如他都具有相同的 source ID，Des ID，LUN 等等

<img src="flow.PNG" width=75%>

# Flow Vision Operation
Flow Vision 监测的对象是 flow, 一般有下面操作
1. 通过指定上面所说的这些参数，create flow
2. 指定 flow 的属性，是 monitor IO, 是 copy 流量，还是 generate 流量
3. Activate flow
4. 显示出收集到的 data

<br/>

## **Flow Monitor**
Flow monitor 最为直观，主要是monitor flow上的各种statistics, 包括  
- Frame statistics (比如 frame count & rate)
- Throughput statistics (byte/sec)
- I/O statistics (I/O count, IOPS)

### e.g.
监测特定端口上 SCSI frame 的数量

> flow --create scsicsflow -feature monitor -egrport 9 –frametype scsicheckstatus


监测端到端（某个 initiator - target）的 throughput

> flow --create endtoendflow -feature monitor -ingrport 5 -srcdev 0x010500 -dstdev 0x040900 –bidir


监测经过某条路径上的指定 LUN 的 I/O

> flow –-create lunFlow1 -feature monitor –ingrport 5 -srcdev 0x010502 -dstdev 0x030700 -lun 4

<br/>

## **Flow Generator**
可在 Fabric 中配置多个 sim port, 并在 sim port 之间模拟流量，从而不需要连接真实的设备。

[^_^]:
    {% asset_img flow-generator.PNG" %}
<img src="flow-generator.PNG" width=70%>

> flow –-create flowCase1 –feature generator -ingrPort 1/1 –srcDev 0x040100 –dstDev 0x050200

<br/>

## **Flow Mirror**
把指定 flow 上的流量 mirror 一份出来，不影响原来的流量，类似抓包。
### e.g.
在 fc device 0x040500 和 0x010200 之间的 通过 port 1/5 的双向流量 mirror 下来 

> flow --create flow_slowdrain -feature mirror –egrport1/5 –dstdev 0x040500 –srcdev 0x010200 -bidir

然后根据 SCSI 的 request/response 情况判断 slow drain device.


