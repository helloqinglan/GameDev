随着近两年来Service Mesh的概念越来越流行，以及越来越多的人涌入这个领域，相应的我也看到越来越多的人对相关技术的迷惑，不知道如何区别这些概念。

我在今年7月份的时候曾在Twitter上发了几条推文，基本上也简要概括了这些概念之间的关系：

```
service mesh confusion #1: linkerd ~= nginx ~= haproxy ~= envoy. None of them equal istio. istio is something else entirely. 1/
```

```
The former are just dataplanes. By themselves they do nothing. They need to be configured into something larger. 2/
```

```
istio is an example of a unified control plane that ties the pieces together in a coherent way. different layer. /end
```

这几条推文中提到了几个不同的工程：Linkerd, NGINX, HAProxy, Envoy和Istio，但更重要的是介绍了Service Mesh data plane和control plane的基本概念。接下来我会从更高的层面介绍下data plane和control plane的含义，以及这些概念与上面提到的这些工程的关系。

### Service Mesh到底是什么
![2a660bea2fefb26e1b9a596e585733de.png](https://cdn-images-1.medium.com/max/600/1*eArKroKLGiZJVFEnTzwXVw.png)

上图展示了Service Mesh的最基础概念。
我们有四个服务 Service A-D，每个服务实例都有一个Sidecar proxy与之对应，所有的网络消息包括HTTP、REST、gRPC、Redis等都从他自己的sidecar proxy进行收发。服务实例自己并不关心网络情况，他只关心自己的sidecar proxy。这样做以后，分面式系统的网络对服务的开发者来说就完全透明了。

### the data plane
在Service Mesh中，sidecar proxy要完成以下功能

- 服务发现：其他的上下游服务都在哪里
- 健康检查：其他的上下游服务是否都正常
- 路由：当本地服务向/foo发起一个REST请求时，sidecar proxy需要知道将该请求发到哪里去
- 负载均衡：如何确定将请求发到哪个实例，以及请求超时的设置，失败的时候是不是要重试，等等
- 认证和鉴权：如何进行身份验证，是否有权限调用指定接口
- 监控：记录每个请求的详情，日志，调用耗时等，用于全链路追踪以及问题分析

以上所有功能都是Service Mesh data plane的职责。
实际上，sidecar proxy就是data plane，换句话说，data plane负责分发、转发和监控服务实例的所有进出请求。

### the control plane
sidecar proxy data plane提供的网络层抽象看起来非常强，但是，proxy是如何知道怎样做路由选择的？服务发现又是怎样实现的？负载均衡、超时设置、熔断等设置又是如何指定的？蓝绿部署以及负载转移又是如何实现的？认证和授权如何进行设置？

上面所有这些项目都是由Service Mesh control plane来负责。
control plane由一组独立的无状态sidecar proxy组成，这些代理构成了一个分布式系统。

我觉得大部分技术人员对划分data plane和control plane感到迷惑的原因是，他们对data plane比较熟悉，而对control plane则比较陌生。我们对使用物理网络路由器和交换机已经非常熟悉了，我们知道一个数据包或请求要从A到达B，能够使用硬件和软件来实现。而我们在这里看到的软件代理实际上是我们使用过的各种代理的一个新版本而已。

![e861b93658b540b4c402ded86d721621.png](https://cdn-images-1.medium.com/max/800/1*mfXDQ8I6hzi8P2fy15sGFw.png)

对于control plane我们实际上也已经用了很长时间，比如对于大多数网络管理员来说他们并不认为这与技术有任何关系。原因很简单，因为我们今天对control plane的使用太常见了。

比如上图展示的human control plane，在某个开发过程中，操作员需要发布一些配置，一般来说他们会使用一些脚本工具来做这件事，使用某类定制的程序来将配置发布到代理服务器上。这些代理服务器类似于我们上面描述的data plane层的角色。

![19b21ab518d43543fce2b862bfd02717.png](https://cdn-images-1.medium.com/max/800/1*jaAyLC0XPN_CDIBV9wG_WA.png)

这张图则展示了在Service Mesh中的control plane操作流程。它由以下各部分组成：

- The human：操作者
- Control plane UI：操作界面，操作者使用UI界面来进行操作，这可能是一个web portal入口，一个CLI交互程序，或者其他某种类型的接口。通过这个管理界面，我们能完成上面列举的control plane需要完成的那些功能，比如蓝绿发布，权限配置，路由配置等等
- Workload scheduler：服务运行在由某种类型的调度系统所构建的基础设施之上，比如kubernetes或者Nomad，调度器负责引导服务及他附带的sidecar proxy
- Sidecar proxy configuration API：sidecar proxy会不断的从各个系统组件中动态的获取状态，整个系统由所有正在运行的服务实例和sidecar proxy构成。Envoy的universal data plane API就是一个具体应用的示例。

总的来说，control plane的目标是设置data plane制定的策略。越是强大的control plane越是能抽象更多的控制系统，为操作人员减轻更多工作。

### Data plane vs. control plane summary
- Service mesh data plane: 处理系统中的所有进入请求，负责服务发现，健康检查，路由，负载均衡，认证和授权等监控等功能。
- Service mesh control plane: 为网络中的所有运行的data plane提供策略和配置功能，并不会处理系统中的请求数据。Control plane将所有的data plane变成分布式系统。

### Current project landscape
有了上面的解释，我们再来看看当前service mesh相关的项目：
- Data planes: Linkerd, NGINX, HAProxy, Envoy, Traefik
- Control planes: Istio, Nelson, SmartStack

我们就不深入探讨每一个项目的实现细节了，只说说为什么会让大家感到困惑的一些问题点。

Linkerd是第一个service mesh data plane项目，它发布于2016年初，Linkerd做了很多让人激动的探索性工作，并且实现了service mesh的设计模式。
6个月后Envoy出现，虽然Lyft在他们内部采用是Envoy是2015年末。
Linkerd和Envoy是在讨论service mesh时最常提到了两个项目。

Istio在2017年5月的时候被提出，istio项目的主目标就是我们在上图中所描述的，一个高级的control plane。Istio默认采用的代理是Envoy，也就是说，istio是control plane，Envoy是data plane。在一段时间里，Istio取得了很多关注，其他的data plane组件也开始将自己集成到Istio中以替换Envoy，比如Linkerd和NGINX。Control plane可以对接多个不同的data plane也事实也逐渐显现，control plane和data plane不需要紧偶合。Envoy提出了universal data plane API则更进一步的将control plane和data plane的桥梁打通了。

Nelson和SmarkStack则进一步展示了contro plane和data plane的区分。Nelson使用Envoy作为代理，然后自己基于HashiCorp stack实现了control plane。SmartStack则大概是第一个实现新的service mesh架构的。SmartStack基于HAProxy或NGINX构建了control plane，然后又展示了service mesh control plane和data plane的分享的可行性。

Service mesh微服务的网络空间正在获得更多的关注，越来越多的项目和开发商开始进入这一领域。接下来的几年里，我们还将会看到更多的data plane和control plane项目，以及各种组件的混合项目。最终的目标会让微服务的网络对操作者来说更加透明。

### Key takeaways
- Service mesh由两个不可缺少的组件构成：data plane和control plane
- 大部分人其实对control plane都比较熟悉，有时候你自己就扮演了control plane的角色
- 各个data plane组件在功能，性能，可配置性和可扩展性方面进行竞争
- 而各个control plane组件则在功能，可配置性，可扩展性和使用便捷性方面进行竞争
- control plane可能会有一个好的抽象API，这样多个不同的data plane都能接进来
- 

https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc