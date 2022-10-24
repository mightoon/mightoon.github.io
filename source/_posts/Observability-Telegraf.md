---
title: "[Observability] 为什么需要 Telegraf"
date: 2019-11-21 10:50:44
description: 一个正在进行的项目，需要把数据中心里所有的虚拟化平台 infrastructure 纳入到 Prometheus 监控，找了一圈没有合适的 exporter.  了解到 Telegraf 有针对 vSphere 的插件，刚好能用上。同时对 TICK 生态稍作延展性研究，整理成本篇文章。
categories: Observability
tags: [Prometheus, Monitoring]
---


说起 Telegraf，其实早些时候在监控选型时有所了解，当时觉得作为数据收集端，没有 Prometheus 生态下的 exporter 用得普及，grafana 上也只有寥寥的 dashboard 可以选择，所以没有太多考虑 Telegraf. 直到后来，需要把数据中心虚拟化平台也纳入到 Prometheus 监控，而针对 VMware vSphere 的 exporter 并不是太成熟，反而在 Telegraf 的 plugin 中找到了对 vSphere 的支持，于是重新调研了 Telegraf，确实有不少可圈可点之处。

## TICK 生态
Telegraf 的并不是单纯的数据收集工具，实际上是 InfluxData 推出的开源生态的一个部分。TICK(Telegraf, InfluxDB, Chronograf, Kapacitor) 共同组成了包含数据的收集、存储、分析和展示的解决方案，是一个可以对标 ELK 的选择。

> Telegraf, a server-based agent, collects and sends metrics and events from databases, systems, and IoT sensors.

从引用的官方描述可以看到，除了采集应用系统的指标数据，Telegraf 的采集源还可以包含数据库或者 IoT 的各种传感器。不同于 Prometheus 的 exporter 各自为阵，Telegraf 以一套代码，通过插件式框架，支持对各种监控对象的数据采集，以及支持多种数据的送出的目的地。


## Telegraf 与 Prometheus 的兼容
从 Telegraf 在 TICK 中的定位可以看出，它的初衷是面向 InfluxDB 生态的，采集的数据推送给 InfluxDB 最为舒服，而对接 Prometheus，虽然也是一个常用的场景，但是从原理上，可能会存在一些小问题。

### 数据格式的问题
Telegraf 会把收集到的数据序列化为 Prometheus 的格式，这个过程通常是将 measurement name 和 field key 拼接，作为 Prometheus 的指标的名字，而 Prometheus lable 名称则对应收集到的数据中的 tag.

而有些 input 插件，会收集 string 类型的指标数据，这对于 influxDB，原本没有问题，但面对 Prometheus 的时候，这类数据需要专门的转换 (string_as_label) ，或者直接丢弃，因为 Prometheus 接受的数据格式为 float64.  

### 标签的问题
对于同一个监控项，Telegraf 采集到的数据，同一个标签下面可能会有多个值，这在 influxDB 上并不是一个问题，但是同为时序数据库的 Prometheus，会严格通过标签的组合来区分不通的时序，同一个标签的不同的值，会被 Prometheus 识别为不同的监控项，造成数据的采集意图的不配备。所以，有时候不得不通过特定的配置，将多值的标签去掉。


## Pull 与 Push
 Prometheus text exposition format.



## beyond collector
There are four distinct types of plugins:

Input Plugins collect metrics from the system, services, or 3rd party APIs
Processor Plugins transform, decorate, and/or filter metrics
Aggregator Plugins create aggregate metrics (e.g. mean, min, max, quantiles, etc.)
Output Plugins write metrics to various destinations


## example