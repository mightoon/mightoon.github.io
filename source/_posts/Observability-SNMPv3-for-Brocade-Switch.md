---
title: "[Observability] SNMPv3 实现 FC 虚拟网络监控"
date: 2022-3-2 09:13:02
description: 在 Prometheus 生态中，往往用 SNMP exporter 监控数据中心的网络，无论是 Ethernet 网络，还是 FC Fabric, 结合相应的 MIB 库， SNMPv2 通常能够满足大多数需求，而且配置简洁，落地简单。我们在一个传统数据中心环境中，用 SNMPv2 监控 FC Fabric, 却发现无法获取 uplink 端口的性能指标数据，通过排查发现 SNMPv2 对 FC 虚拟网络的支持不足，这里总结一下出现问题的原因和处理的方式。
categories: Observability
tags: [SNMP, Prometheus]
---




## FC Fabric 监控的问题
在该传统数据中心环境中，客户使用多台中端商业存储支持应用业务，并通过 FC 网络传输数据。客户的不同业务被划分在不同的 `FC Virtual Fabric` 中，相互隔离。

我们在 edge switch 面向各个业务主机的端口收集性能数据，同时也在 uplink 端口收集数据以统计业务总量，结果发现面向业务的主机端口数据能正常收集，但是 uplink 端口没有数据。

<img src="topo.png">

## FC 网络背景知识
这里不得不提到一个 FC 网络的背景知识。根 Ethernet 网络划分 VLAN 类似，一个实体的 FC 网络也常常被划分为多个虚拟网络，每个虚拟网络用独立的 FID 区分，用以隔离不同的流量，以及简化网络的管理。具体到交换机上，会在 FC 网络中的每一台物理 FC 交换机上划分多个逻辑交换机，分配相应的 FID，整个 FC 网络中只有相同 FID 的逻辑交换机，才能相互通信。

为了跨物理交换机通信，当然可以把各物理交换机上的相同 FID 的逻辑交换机连起来，但是如果一个物理交换机划分了多个逻辑交换机，而他们分别需要与另一台物理交换机上的多个个逻辑交换机通信，那么需要多条不同的连线：

<img src="multi-link.png">

为了简化这个问题，FC 网络中引入了一种特殊的逻辑交换机，叫做 `base logical switch`, 类似于 ethernet 的 trunk port，可以将同一个物理交换机上所有的逻辑交换机的流量聚合，统一发送接受，并且能力区分：

<img src="base-link.png">

对比文章开头的拓扑，客户环境部署的就是 base switch 这种模式。

## SNMP 协议对 Virtual Fabirc 的支持
`SNMP` 作为一个很成熟的协议，一直活跃在网络监控、远程配置、网络管理等方面。早期的 V1，V2 版本，简单地通过 SNMP 管理站点（NMS）向 SNMP 代理（Agent）发送查询请求来获取数据。 V3 中加入了身份验证和加密处理的，在 NMS 于 Agent 的交互中，需要加入更多的安全方面的参数。

<img src="snmp-version.png">

而 Virtual Fabric 的环境，需要升级到 SNMP v3 才能够支持
```
NOTE:
SNMPv3 must be used to manage a Virtual Fabric

When an SNMPv3 request arrives with a particular user name, it executes in the home Virtual Fabric. 
From the SNMP manager, all SNMPv3 requests have a home Virtual Fabric that is specified in the contextName field. 
When the home Virtual Fabric is specified, it is then converted to the corresponding switch ID and the home Virtual Fabric is set. 
If you do not have permission for the specified home Virtual Fabric, this request will fail with an error code of noAccess. 
```

## SNMP v3 在监控系统中的实现
### Switch 上 enable SNMP v3
一般来说，在主流的企业级交换机上，SNMP v3 需要主动 enable. 
对于性能的监控，read only (ro) 的权限就够了。 


```
    Switch1:FID20:user1> snmpconfig --show snmpv3
    User 5 (ro): user1
            Auth Protocol: noAuth
            Priv Protocol: noPriv
            Engine ID: 00:00:00:00:00:00:00:00:00

    Switch1:FID20:user1> userconfig --show user1
        Account name: uesr1
        Description:
        Enabled: Yes
        Password Last Change Date: Tue Apr 16 2020 (UTC)
        Password Expiration Date: Not Applicable (UTC)
        Locked: No
        Home LF Role: basicswitchadmin
        Role-LF List: basicswitchadmin: 1-128
        No chassis permission
        Home LF: 20
        Day Time Access: N/A
```


### SNMP Exporter
以用户 user1 向 SNMP agent 发起请求，跨越 virtual fabric，取到 base switch （FID 15）的性能指标。

根据前面 Note 中的说明，user1 工作在它的 home fabric，也就 FID 20，要跨越到 FID 15 访问 uplink，应该需要一个参数。查看 exporter的[源码](https://github.com/prometheus/snmp_exporter/blob/74f0ae5162f0e373f5da0e6dda0c6feb05b1153d/config/config.go)，找到这个参数叫做 `context_name`

```go
        type Auth struct {
                        Community     Secret `yaml:"community,omitempty"`
                        SecurityLevel string `yaml:"security_level,omitempty"`
                        Username      string `yaml:"username,omitempty"`
                        Password      Secret `yaml:"password,omitempty"`
                        AuthProtocol  string `yaml:"auth_protocol,omitempty"`
                        PrivProtocol  string `yaml:"priv_protocol,omitempty"`
                        PrivPassword  Secret `yaml:"priv_password,omitempty"`
                        ContextName   string `yaml:"context_name,omitempty"`
        }

```

加到 SNMP exporter 的配置文件中：

``` yaml
    brocade_mib:
      version: 3
      auth:
        username: user1
        security_level: noAuthNoPriv
        context_name: "VF:15"
```


<br>

### Prometheus Jobs
在 Prometheus job 中添加 SNMP 数据收集的 job

```yaml
    - honor_timestamps: true
      job_name: snmp_fc
      metrics_path: /snmp
      params:
        module:
        - brocade_mib
```
<br>

经过以上配置，uplink 上的流量终于可以取到：

<img src="traffic.png">

## References
1. [Prometheus SNMP Exporter](https://github.com/prometheus/snmp_exporter)
2. [SNMP v2 vs. V3](https://askanydifference.com/difference-between-snmpv2-and-snmpv3/)
3. [Brocade Fabric OS Administration Guide](https://docs.broadcom.com/doc/FOS-82x-AG)