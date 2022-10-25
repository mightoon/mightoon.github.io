---
title: "[Observability] 为什么需要 Telegraf"
date: 2019-11-21 10:50:44
description: 一个正在进行的项目，需要把数据中心里所有的虚拟化平台 infrastructure 纳入到 Prometheus 监控，找了一圈没有合适的 exporter.  了解到 Telegraf 有针对 vSphere 的插件，刚好能用上。同时对 TICK 生态稍作延展性研究，整理成本篇文章。
categories: Observability
tags: [Prometheus, Monitoring]
---


说起 Telegraf，其实早些时候在监控选型时有所了解，当时觉得作为数据收集端，没有 Prometheus 生态下的 exporter 用得普及，Grafana 上也只有寥寥 dashboard 可以选择，所以没有太多考虑 Telegraf. 直到最近，一个项目需要把虚拟化平台也纳入到 Prometheus 监控，而针对 VMware vSphere 的 exporter 并不是太成熟，反而在 Telegraf 的 plugin 中找到了对 vSphere 的支持，于是重新调研了 Telegraf，确实有不少可圈可点之处。

## TICK 生态
Telegraf 的并非单纯的数据收集工具，至少 Telegraf 志不仅在此。实际上 InfluxData 推出了一整个开源生态：TICK(Telegraf, InfluxDB, Chronograf, Kapacitor) 共同组成了包含数据的收集、存储、分析和展示的解决方案，这是一个可以对标 ELK 的选择。而 Telegraf 和 InfluxDB 无疑是其中相比于后两者更为重要的两个部分。 

> Telegraf, a server-based agent, collects and sends metrics and events from databases, systems, and IoT sensors.

从引用的官方描述可以看到，除了采集应用系统的指标数据，Telegraf 的采集源也包含数据库或者 IoT 的各种传感器。不同于 Prometheus 的 exporter 各自为阵，Telegraf 以一套代码，通过插件式框架，支持对各种监控对象的数据采集，以及支持多种数据处理的下一站。

{% img https://www.influxdata.com/wp-content/uploads/Influx-1.0-Diagram_04.20.2020v2.png 640 TICK %}

## Telegraf 与 Prometheus 的兼容
从 Telegraf 在 TICK 中的定位可以看出，它的初衷是面向 InfluxDB 生态的，采集的数据推送给 InfluxDB 最为舒服，而对接 Prometheus，虽然也是一个常用的场景，但是从原理上，可能会存在一些小问题。

#### 数据格式的问题
Telegraf 会把收集到的数据序列化为 Prometheus 的格式，这个过程通常是将 measurement name 和 field key 拼接，作为 Prometheus 的指标的名字，而 Prometheus lable 名称则对应收集到的数据中的 tag.

而有些 input 插件，会收集 string 类型的指标数据，这对于 InfluxDB，原本没有问题，但面对 Prometheus 的时候，这类数据需要专门的转换，或者直接丢弃，因为 Prometheus 接受的数据格式为 float64. 如果 string 对于指标而言是有意义的，通常有两个方式转换成 Prometheus 识别的格式，一是通过配置 `string_as_label`, 更稳妥的方式是通过 Telegraf 提供的 processor 插件 `converter`

> The converter processor is used to change the type of tag or field values. In addition to changing field types it can convert between fields and tags.


#### 标签的问题
对于同一个监控指标，Telegraf 在采集过程中有可能会添加标签，而这个标签被允许有多个值，这在 InfluxDB 的存储模型中，能够得到正确的处理，但是同为时序数据库的 Prometheus，会严格通过标签的组合来区分不同的时序，因此同一个标签的不同的值，Prometheus 识别为不同的监控项，这样会隐形地造成与数据的采集意图的偏差。所以，这种时候需要通过 Telegraf 特定的配置，将多值的标签去掉。


## Pull 与 Push
Exporter 遵从 Prometheus 的 pull 架构，将抓取的指标通过类似 /metrics 这样的 endpoint 暴露出来，供 prometheus 获取。而 Telegraf 的 output 插件的逻辑，往往依循 push 模式，把数据推到比如 InfluxDB. 为了适应 Prometheus 的模式，Telegraf 通过实例化一个 [prometheus_client](https://github.com/influxdata/telegraf/tree/master/plugins/outputs/prometheus_client) 来实现。

Pull 与 Push 孰优孰劣，取决于应用场景，业务逻辑，以及所处的环境，一些明显的区别在于：
- Pull 模式是监控系统主导的，便于在监控方集中配置，而 Push 是被监控的应用系统主导的，需要在各个监控目标上配置。
- Pull 便于监控系统决定以何种频率抓取数据，这对监控系统自身是一个保护，不容易过载。Push 通过推送传递数据，必要时需要在多个被监控目标之间协调数据推送的节奏。
- 如果监控系统和被监控目标处于不同的网络安全区域，Pull 可能无法从外界进入防火墙，而 Push 流量从防火墙内部往外，一般不受限制。
- 对于执行时间相当短的任务，当短于抓取数据的时间间隔，Pull 系统可能无法获得响应的指标，而 Push 在这一点上有比较大的自主权，这也是 Prometheus 为何提供 Push Gateway 的原因。


## Telegraf 的工作方式
最后，说一下 Telegraf 的工作方式。可以简单地归结为 `plugin-driven`.  数据流过 Telegraf 的 plugin system，由四类主要的 plugin 处理：

{% img https://www.influxdata.com/wp-content/uploads/Main-Diagram_06.01.2022v1.png Telegraf-plugin-system %}

- Input Plugin:  收集数据
- Processor Plugin:  转换，修饰，以及过滤数据
- Aggregator Plugin： 聚合数据
- Output Plugin： 对接后续数据处理或存储组件

开篇说到 Telegraf 的志向不限于仅仅对数据的收集，实际上，从 plugin 的框架可以看出，Telegraf 极力丰富自己在数据清洗、数据转换、以及一些类似标签分类等数据准备方面的工作，结合自家 InfluxDB 于 TSDB 中的地位，在时序领域形成一道不容忽视的力量。
