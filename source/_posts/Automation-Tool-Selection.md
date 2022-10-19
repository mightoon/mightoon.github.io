---
title: "[DevOps]自动部署管理工具的选择"
date: 2017-9-3 16:10:15
description: 随着数据中心设备的增加和对效率的要求，部门提出了对设备的自动化管理需求。为此，特意比较了主流自动化部署和管理的工具，本文借此梳理一下包括 func, puppet, chef, saltstack 等诸多种工具的设计思路和发展轨迹。 
categories:
- DevOps
tags:
- Automation
- DevOps
---

 > **Working in IT, you're likely doing the same tasks over and over. What if you could solve problems once and then automate your solutions going forward?**


## 氛围
还得从DevOps运动说起。  
DevOps运动的出现是由于软件行业开始清晰地认识到，开发、运营和质量保障工作必须更加紧密协作，才能保证交付和维护。作为Agile的延伸，DevOps把Ailge依靠Dev&Biz合作，通过增量交付的方式来解决需求多变难题的经验进一步延伸到IT运维，通过Dev&Ops的紧密协作提高软件交付的质量和频率。打开DevOps的正确姿势需要充分理解DevOps的原则，分析项目自身需求和条件，选择方法和流程，构建适当的工具，最终实现full stack的自动化。

## 混战
是的，DevOps需要工具，今天讨论的正是DevOps运动的参与者。  
早期的部署和配置管理分别是通过不同的工具完成，比如部署工具Cobbler, Kickstart, 配置管理工具CFEngine, Puppet等，他们共同的特点是`复杂`: 安装复杂，配置复杂，使用复杂。即便如此，当年这些工具依然各自流行于自己的时代，有些直到现在。敏锐的部署管理者发现，这些工具间的配合存在一个空隙，即如何高效执行临时任务，于是RedHat的大牛们开发了一个SSH分布式框架Func.

另一条线上，Chef已然崛起。"不会Ruby的攻城狮不是好的厨子”，而Chef是一个好的厨师。Chef对配置管理任务精确动刀(knife), 服务器上通过菜谱(cookbook)传递部署方案，并在客户端秘方(recipes)中记录做法。配置管理如烹小鲜。Chef具备优秀的架构，几乎能解决任何配置问题，但是一直以来叫好不太叫座，因为Chef要求配置管理人员熟悉Ruby, 学习曲线陡峭，模块开发难度大。中心服务端需要依赖于CounchDB, RabbitMQ, Solr, Java, Erlang等软件，配置过程繁琐，于是一定程度上被局限在`具备编程思维`的管理员小圈子中。

面对Chef的竞争，Puppet当仁不让，高手之争，已不在招式本身，在于细微之处。虽然同样是基于Ruby, Puppet并不要求用户懂太多的Ruby, Puppet的配置语言更高级和通用；同样采用server-agent模式，Chef的服务器端的配置放在CouchDB和Solr索引等二进制文件中，而Puppet服务端的配置是文本文件，这样易于发布、备份和扩展，也更符合unix系管理员的使用习惯。Puppet的理念源自CFEngine, 更加合理的实现，加上常年较高的开发活跃度，Puppet已然被许多大公司所接受。当然，Puppet并不完美，比如C/S架构相对比较重量级，比如它偏配置管理而弱于命令执行，更重要的是，Puppet可以再简单一点，这为后来的颠覆者们留下了机会。

## 突围
Michael DeHaan, 这是一个应该被记住的名字，他是Func的作者之一，在吸收Func优秀理念的基础上，同时结合在Puppet Lab的工作经验，正着手开发了一个配置管理工具。Michael认为Puppet和Chef虽然强大，但他们都基于自有的一套机制，加上配置管理和程序发布过程过于复杂，往往超过了一般公司的需求，容易被运维团队排挤。新工具的定位是集`配置管理`，`应用部署`，`临时任务`于一身，提炼出同类工作中最核心的功能，让有经验的运维人员能够在15分钟之内学会，并用自己熟悉的语言扩展。没错，这就是Ansible的诞生。API深受Func的影响；playbook结合Chef定义执行顺序和Puppet事件机制的优点；Push构架为多节点部署而生同时又抛弃中心节点的概念；提倡“提前崩溃”避免问题在依赖链上传递；可以用任何语言编写扩展插件只要能够返回JSON格式。Ansible的palybook不是一门编程语言而是一种类似于自然语言的数据结构，这意味着运维工作不再是一项开发型任务。Ansible本身简洁而小巧，并汇聚了一批活跃的社区贡献者，使得它更加健壮可靠。

罗列一下Ansible的几个特点：
* Agentless
* Stupied Simple
* SSH by default
* YAML no code
* Idempotent
* Easy extension

Ansible凭借先进的理念和成熟的框架对Chef, Puppet构成巨大威胁，但Ansible也并不是孤独的存在，它的对手，叫做Saltstack.

## 对手
Saltstack胜在执行效率，基于ZeroMQ的传输方式让Salt比基于SSH的Ansible快出一个数量级，特别是在大规模部署上。Ansible的应对方式是引入fireball模式，通过SSH临时建立起ZeroMQ的传输通道，使得在大量命令分发的任务中效率与Salt难分伯仲。如果吹毛求疵地认为Ansible建立ZeroMQ需要额外的时间，亦如果任务类型总是需要往大量服务器同时分发大量命令，Salt也许仍然是更好的选择。而事物的两面性在于ZeroMQ并不具备传输加密能力，所以Salt必须依靠自身的AES implementation，这当然逊于SSH天然的安全性，即便是在fireball 模式下，Ansbile借用Keyczar也更值得信赖。

Salt通过daemon能更好的控制异构的系统，分发的命令不依赖于目标server上的执行环境。同样受益于daemon的精巧设计, Salt的维护和更新简单得无以复加，从而适应Salt相对快速的迭代。另一边Ansible通过daemonless成功避免了`managing the management`的问题，但是daemonless也并非没有弱点，典型见诸windows这片Ansible还未完全征服的疆土。此外，没有daemon也让Ansible在每次推送命令的时候除了control data本身，不得不携带module的信息，虽然已经非常的轻量级。  Deamon与daemonless，源于两种不同的设计逻辑，看似重大差别，实际殊途同归，如果一定分个高下，那也是见仁见智。

Salt和Ansible的执行均以Push为基础，而Slat的requisite机制让server对彼此有所感知，在赋予执行过程极大的自由度的同时，也使得结果在一定程度上失去预见性，相同的配置脚本对同一组目标服务器的执行可能得到不同结果。因此，实时性处理那些在执行中需要同步信息的任务是Salt比较擅长的。Ansible以脚本定义的执行顺序推送，简明和直接，辅之于简单事件驱动和handler，配合“提前崩溃”机制，让执行结果可靠，稳定。  
名如其人，Salt拟定规程(state), 设想各种情况以及应对，让执行者在任何情况下有章可循。Ansible犹如一个导演，提供剧本(playbook), 监督拍戏并随时叫cut, 保证演绎出一个层层递进，艺术完整的剧情。Salt和Ansible在执行顺序上坚持不同的逻辑，源自对系统管理和资源依赖不同的理解，而这背后，隐约能够看到前辈Puppet和Chef的影子。

行文至此，大概明白了对自动部署工具的选择。
