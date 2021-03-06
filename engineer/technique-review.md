从2017年下半年开始，在解决了服务器的崩溃、假死等问题后，开始思考如何能让服务更稳定，同时在出现问题后能够更快的定位到原因。

也因为技术栈的转变，使得我有机会放下前十多年的C++经验，转为用Java开发的眼光来看待技术的发展；对于应用开发来说，Java无疑是一块更为成熟的领域，尤其是互联网公司在这个领域内的技术积累已经走了很远，相比起来，C++游戏开发领域的技术发展却要滞后很多。

尤其是互联网公司对技术的开放态度，和对开源社区的建设力度；鼓励技术的传播与交流，使得互联网基础技术越来越完善，同时也越来越多样化，能适应各种不同的需求。

对线上业务的优化改进必须循序渐进，力求稳妥；用一个大家常用的类比：是在给一辆高速行驶的汽车换轮胎。但是因为欠下的技术债太多，如果不补上，那始终会是在头痛医头、脚痛医脚的修修补补，不如早下决心。

以下是在这个过程做过的一些比较典型的工作，始终遵循的是“有节制的进行基础设施的抽离，有节制的引入新技术与框架”的原则。

在总结的过程中又让我想到了以前读过的那本《Unix编程艺术》，并且拿出来重温了一遍。其中总结的Unix哲学：
```
一个程序只做一件事，并做好。程序要能协作。程序要能处理文本流，因为这是最通用的接口。
```
不论是最初的监控平台：Metrics + InfluxDB + Grafana，日志平台：ElasticSearch + LogStash/Filebeat + Kibana，还是后来的容器化与集群调度：Docker + Kubernetes，以及持续集成：Jenkins CI/CD Pipelines里的那一个个串连起来的组件，无不体现了“一个程序只做一件事，并做好；通过功能组合来实现复杂功能”的思想。

一言以蔽之，就是K.I.S.S.原则。

以及Unix历史教给我们的最大的规律：**“距开源越近就越繁荣”**。

### 一、监控平台
```
监控系统应该解决两个问题：什么东西出故障了，以及为什么出故障。 ---- 出自Google SRE
```
Google总结的监控系统的四个黄金指标：
- 延迟
- 流量
- 错误
- 饱和度

如果不知道该监控什么时，那就监控这四项指标
#### 1) 方案
Metrics + InfluxDB + Grafana

监控系统是技术升级所迈出的第一步。

在经历了一段时间低效的的bug查找之后，迫切的需要一个工具来告诉我们系统里到底发生了什么，而监控系统正是因此而生。

2017年年底的时候[对当时找到的一些监控指标库和监控系统做了一个对比](
https://app.yinxiang.com/fx/65f242b4-e644-476e-b5f8-7b7a1b28d8f8)，最终选定了Metrics + InfluxDB + Grafana的方案。从方案里也能看到，主要在于Prometheus和InfluxDB(+Metrics)的选择。

今天再回过头来看当时的选择也是没有问题的，Prometheus作为CNCF的第一个成员组件，无疑是优秀的，如果放在今天，我会毫不犹豫的选择Prometheus；但在当时，应用与服务的概念还完全没有，Kubernetes的引入也在很久之后，而Prometheus是为云原生环境的监控而生的，此时直接上Prometheus会很困难。

![img](https://prometheus.io/assets/architecture.png)
```
Effective use of Kubernetes requires architects to think about applications and workloads rather than machines and processes.
Monitoring applications deployed to Kubernetes requires the same shift in thinking.
```

不过这个调研里有个错误：

zipkin, pinpoint不应该与Metrics做对比，他们都属于分布式系统的调用链追踪组件，同类型的还有jaeger等。

围绕tracing还有一个开源项目叫OpenTracing，试图把各种追踪库做接口统一，这样具体的实现可以替换。在我们后来的实践中也曾尝试过OpenTracing + Jaeger的方案。

#### 2) 改进
为什么说放到今天会选择Prometheus而不是InfluxDB，除了Kubernetes环境已经运行较长时间，对应用与服务的概念有了一定理解外，在使用InfluxDB时遇到的两个不太方便之处也有一定原因：
1. Prometheus的“拉(Pull)”模式要优于InfluxDB的“推(Push)”模式

在InfluxDB(+Metrics)的推模式下，我们封装了一个库用来做监控，这个库使用Metrics搜集应用的公共指标，同时提供接口给应用添加自定义指标，然后使用influxdb reporter定时将指标发送到InfluxDB。

所以应用就需要有一个配置来指定InfluxDB的地址，以及这个应用需要监控哪些公共指标，而且这个配置在不同的环境下还不一样。当然，在Kubernetes环境下可以配置一个external service来做中转，这样也能实现配置完全一致，但总归来说是个比较麻烦的点。

但是Prometheus的拉模式则不存在这个问题，每个应用只需要实现监控接口，不用关心监控数据会发往哪里。由监控服务Prometheus自己通过服务发现找到这些服务，并调用服务的metrics接口去拉监控数据。

本质上来说**配置并没有减少，但这是一个谁依赖谁的问题**。

我觉得，配置应该由服务自己来负责，所以我更倾向于Prometheus的拉模式。

2. InfluxDB的负载问题

因为InfluxDB是商业产品，开源版本只支持一个实例，当监控项越来越多，以及用于报警的持续查询越来越多的时候，单实例总会遇到瓶颈。

当然可以像携程做的那样，在开源版本之上自己来实现水平扩展，把负载分担到多个实例，但是Prometheus直接就支持多实例与水平扩展。


InfluxDB与Prometheus并不是一个只能选其一的方案，事实上不论是使用推模式还是拉模式，获取监控数据的库可以完全一样，甚至可以在自己封装好的库里同时支持推模式和拉模式。

InfluxDB可以使用[Prometheus提供的指标库](https://github.com/prometheus/client_java)，Prometheus也可以将InfluxDB作为自己的存储后端。

这也再次印证了前面提到的Unix哲学：**一个程序只做一件事，通过功能组合来实现复杂功能**。

### 二、配置中心
配置分为业务配置与运维环境配置，在业务体量没有那么大，系统架构并不复杂的时候也就没有必要做区分。

早期业务的配置修改基本上都通过业务的管理后台来实现，但这样给开发带来了一些额外的麻烦：需要在应用上提供修改配置的接口，以及管理后台上制作修改配置的页面。

另外还有一部分配置文件打包到发布包里面，如果需要修改只能运在部署后手工操作，不仅麻烦，而且修改后必须重启服务才能生效。

在充分评估了业界方案后，引入了[携程开源的Apollo配置中心](https://github.com/ctripcorp/apollo)

之后进一步规范了配置的使用，包括
- 本地配置全部迁移到配置中心
- 不需要交互式修改的配置从管理后台迁移到配置中心，管理后台仅保留查询和交互式修改功能

配置中心的技术升级完成后，线上至少一半以上的变更不再需要停服；新增配置项不会给开发人员带来额外工作量；以前只能由开发人员维护的业务配置也可以交由产品和PM去维护。

并且，Apollo配置中心也确实经受住了作为基础服务所要求的稳定与可扩展性考验。在开源版本的基础上我们主要在Portal上扩展了不少功能，配合着在Config Server上也有部分修改，以及同步新版本的修改。上线后服务的升级、迁移过程都比较平稳，未出现过问题。

![img](https://github.com/ctripcorp/apollo/raw/master/doc/images/overall-architecture.png)

#### 思考
虽然配置中心给我们的线上业务带来了不少便利，另外在几乎所有介绍微服务的文章中都能看到“配置中心”这样一个组件，阿里云也提供了[应用配置管理服务ACM](https://www.aliyun.com/product/acm)，但是否应该使用集中式的配置中心在业界仍然存有一定争议。

Google SRE里这样介绍他们的配置管理：
```
将配置文件存放于我们的主要代码仓库中，同时进行严格的代码评审。
项目负责人在分发和管理配置文件时有多种选择，可以按需决定究竟哪种最适合业务。比如：
- 使用主分支版本管理配置文件
- 将配置文件与二进制文件打包在同一个MPM包中
- 将配置文件打包成MPM配置文件包
- 从外部存储服务中读取配置文件，比如Chubby, BigTable或VCS
```

在Uber的智能化运维平台分享中也曾提到，因为集中式配置中心引起过一次较大的线上事故，所以之后逐步改成了独立配置。理由是集中式的配置中心因为每个人都可以访问，环境配置都放在一起，权限控制不当极容易引发事故。

这当然是有道理的。

### 三、服务发现
通过引入[ZooKeeper](https://github.com/apache/zookeeper)做注册中心来实现自动的服务发现。

与其他如监控、配置管理等附加服务的接入不一样，因为引入了ZooKeeper做注册中心，我们移除掉了原有系统中的一个服务进程。这对线上稳定运行的系统来说是非常大的改动，整个过程所以更加的谨慎。

为什么要引入新的注册发现机制？当然不是因为互联网公司都这么做所以要改。

原有的方式对进程启动顺序有较强的依赖，另外对进程的异常退出和重连处理不够好；在出现问题时最后无奈的解决方法是全部进程停掉，再按顺序启动。

当然可以修改原有的实现，让他能够处理各种异常，且不依赖先后顺序，但这样改动所花的时间与引入一个稳定可靠的注册发现系统所花的代价相比如何。

ZooKeeper是较早接入，持续开发时间也较长的一个组件。一方面是确保稳定，一直缓慢的推进，确保每一步的修改都有兜底方案；另一方面也是因为最初不仅仅只是打算用它来做注册中心，还打算将配置中心也放到这里。

因为会需要管理配置数据，也为其开发了一个管理后台。
后来觉得需要开发的东西太多，于是再次进行了评估，并找到了更合适的替代品Apollo，ZooKeeper也就仅保留了服务发现功能。

#### 思考
ZooKeeper可以算得上是最古老的分布式系统基础服务之一，由于Java语言的特性以及太多的第三方依赖，使得ZooKeeper的运行资源占用相比Go写的etcd和Consul要大不少。

另外由于ZooKeeper原本是用来做分布式系统一致性同步的，用它来做注册中心只是因为他提供了对节点数据的Watch功能，但这也需要依赖客户端库来实现，实际应用中一般是[Curator](https://github.com/apache/curator)，这也使得用了ZooKeeper的应用也会变的有点重。

```
ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.
```

除此之外ZooKeeper并无大的劣势。

但是，如果放在今天来做选择的话，[Consul](https://github.com/hashicorp/consul)会是一个更好的选择。([etcd](https://github.com/etcd-io/etcd)因为功能过于单一，要当作注册中心还需要依赖其他第三方组件)

![img](https://www.consul.io/assets/images/consul-arch-420ce04a.png)

而且，Consul的实现方式也更符合“Unix哲学”：
Consul服务实现注册中心功能，Consul agent负责服务的注册与更新通知，应用与agent使用http的文本方式进行通信。部署时可将agent与应用部署在一个pod内，也可以在node上部署一个agent。
这与service mesh的sidecar模式也非常的类似。

### 四、日志中心
集中式日志管理解决的首要问题是高效的处理分散在每台不同的服务器，每个目录下的日志文件。

当然，即使不用ELK，也可以用一个脚本来将分散的日志集中到一起，免去逐个目录检查的麻烦。但是，ELK的优势不仅在于数据聚合，还有全文索引，以及在这之上使用插件来实现的持续查询和报警。

这块工作与开发比较独立，基本上都是由运维人员来完成，也没有花费太多时间，很快就见到了成效。

ELK日志中心一般指LogStash + (Kafka) + ElasticSearch + Kibana。

虽然已经有了基于InfluxDB + Grafana的时序状态数据监控和报警(APM)，但有些状态异常并不会反映到监控指标上，而是一行警告或错误日志，这类异常我们需要通过ELK来做报警。

#### 结果
完成了这些之后，基本上对服务的稳定性能够做到胸有成竹；不用每天主动去关心服务有没有出问题，只要没收到报警，服务就是稳定的。

一旦有报警出现，首先从APM上看各项指标是否正常，然后从ELK上搜索相关进程的日志，绝大多数情况下都能快速定位到问题；大部分的修改也都可以通过配置变更立即来完成，或者通过配置关闭某些存在问题的功能。

如果需要对服务进程进行修复来解决问题，那就是另一个主题的内容了：容器与集群调度。

### 五、容器与集群调度
使用容器部署和集群调度的需求来自两点：
 - 运用Kubernetes集群的能力做到滚动更新，更新过程对用户无影响
 - 推动CI/CD与DevOps，加快产品开发、上线的迭代速度

在将业务迁移到Kuberntes集群之前在内部搭建的集群上先熟悉了比较长时间，包括Docker和Kubernetes的使用，服务的概念，负载均衡的实现，滚动更新的过程等等 。

2018年3月份完成了容器化部署的目标与规划，分为三条线同时推进，最终在半年内落地
- 容器化部署的技术准备：包括服务基于Docker部署、使用Kubernetes进行服务编排的方法，以及从机器与进程向应用与服务思维的转变
- 应用的服务化改造：为了能够使用Docker部署多节点服务，应用需要做到去状态，确保分布式环境下的数据一致性
- CI/CD平台建设：应用要能做到持续集成与持续部署，要有相应的平台能力

并且所有这些工作都要在不影响线上业务的前提下进行，这是需要保守的底线。

#### 思考
回过头来看，我们在没有容器使用经验的条件下，顺利的完成了服务的容器化部署。虽然其间也出现过一些小问题，比如探针设置的问题，导致滚动更新过程中会有服务中断；比如JVM参数与容器环境参数的设置问题，导致应用的重启；也都能比较顺利的找到问题的原因并快速解决。

但是容器技术涉及到的知识点非常多，在实践过程中其实也发现了基础的缺失。

所有悬而未决的问题最后都会成为债务。
加强对Docker和Kubernetes的深度理解，知其然更要知其所以然。

### 六、持续集成
持续集成并不是从容器化部署后才开始引入的一项工作，而是从一开始就在进行。

包括最初为了能够更顺利的使用Jenkins进行自动化打包，开始规范pom文件，定义版本号规范，搭建Nexus包管理平台，代码仓库从svn迁移到Git等。

在完成了这些基础的改进后，首先是使用Jenkins做到了自动化打包，生成好的部署包上传到指定位置，可直接用于部署。

在容器化工作开始后，再一次将持续集成流程进行优化，使用Jenkins CI Pipeline的方式，把一个个步骤抽象成Stage。

![img](https://jenkins.io/zh/doc/book/resources/pipeline/realworld-pipeline-flow.png)

主要部分包括：
- Git -- master分支提交自动触发构建
- stylecheck -- Google的代码格式检查
- Junit -- 单元测试
- SonarCube -- 代码静态检查
- Build - 构建部署包
- Build Notify -- 构建通知，发送钉钉消息

#### 结果
持续集成是软件开发过程中不可缺少的关键环节，通过持续集成能够极大的加快开发速度，让开发人员专注在业务本身，不用去关心重复性的琐事。

另外，统一的构建环境也保证了部署包的安全。为了更加强化安全性，我们在Docker镜像仓库上做了认证与授权，不允许普通用户随意上传关键镜像；在Nexus包管理平台上同样也做了认证与授权，只允许从CI机器上传包；而Git上则有与每次构建版本相对应的tag记录；通过这一系列的方法，确保了运维拿到的每一个包都是安全可靠的，也是可追溯的。

### 七、自动化部署
```
没有几个人能像机器一样永远保持一致。
一致性地执行范围明确、步骤已知的程序--是自动化的首要价值。
```

Google SRE花了大量篇幅来阐述自动化的重要性与实现方法；但是自动化运维不仅仅是工具的使用，更重要的是人的思维转变，以及相应的人员能力升级。

自动化所能带来的价值也是明显的，不仅仅是对业务的贡献，同样重要的也是对人的价值体现；认识到了这个价值，自然也就不会有太多的阻力。

自动化以后也利于对整个流水线工作进行量化。从代码修改提交到部署完成的时间一直在缩短，平均发布间隔时间也同样在缩短。

当然，发布流程还没有实现完全的自动化，依然需要人工干预，这与我期望的目标仍然还有一定距离。

以及整个DevOps自动化平台的建设也刚见曙光，还需要反复验证与迭代；事故跟踪系统也还没有集成到整个DevOps环境中。但至少，自动化走向了正确的方向。

```
发布工程师开发工具，制定最佳实践，以便让产品研发团队可以自己掌控和执行自己的发布流程。
```
这才是最终的目标。

Google SRE有一章叫“简单化”：
```
一个设计良好的分布式系统是由一系列合作者组成的，每一个合作者都具有明确的、良好定义的范围。
```
这也就是我们一直提到的“Unix哲学”。