![Flags](http://farm2.static.flickr.com/1323/900012193_4de6e8d2d4.jpg)

### 一、什么是Feature Flag
"[Feature Flag](https://en.wikipedia.org/wiki/Feature_toggle)"有时也被称作"[Feature Toggle](https://martinfowler.com/bliki/FeatureToggle.html) (2010年)"，可以通过配置的方法将某些功能打开或关闭，而不需要重新部署代码。

目前已知的最早提及使用这项技术的是[Flickr](http://code.flickr.net/2009/12/02/flipping-out/)，他们在2009年公开的文档里声称，最初引入这项技术是为了解决产品快速迭代过程中某些需要长期开发的功能如何在不同的环境下方便的关闭和打开。

Feature Flag功能的配置如下：
![](http://farm3.static.flickr.com/2553/4153769740_d4b45ea2c8.jpg)

Feature Flag对于Flickr实践持续集成非常的有帮助，在2009年的时候Flickr就已经做到了**每天发布多次**。但是Feature Flag也不是万能的，在这篇文章中他们同样也提到了需要限制该功能的使用，否则太多的功能开关充斥在代码中会变成恶梦；另外功能一旦确定上线，需要及时把功能开关和相关代码都删除，避免代码里面判断分支的膨胀。


### 二、Feature Flag与A/B testing
![](https://martinfowler.com/bliki/images/featureToggle/featureToggle.png)

从最初的设计上来说，Feature Flag只是用于支持CI，即允许未完成的代码包含在发布版本中，但是使用功能开关来将其关闭。随着技术的发展，Feature Flag逐渐被用于更多的需求领域，比如**金丝雀发布**和**A/B testing**。

本质上来说，Feature Flag将一些原本通过代码做判断的逻辑改到了通过配置来实现，并且可以在软件运行过程中动态的修改。这类逻辑包括：
- 谁可以看到新的管理界面
- 哪个级别的用户可以解锁这个新功能
- 什么时候开始切换到新的数据库
- 是否需要在按钮前加一个图标来提高转换率

如果在一个应用中包含了大量的Feature Flag，那就需要有好的方法来管理这些Flag，否则将会面临很大的挑战。目前在这方面比较通用的方案是：
- 使用Toggle points表示功能开关
- 多个Toggle points组成一个Toggle router；一个Toggle router表示一个特性组
- 使用Toggle context为Toggle router提供上下文信息，比如用户信息等
- Toggle configuration则为Toggle router提供系统环境信息

### 三、Feature Flag的应用

可以从两个方面来区分一个Feature Flag的类型
- Feature Flag将会在线上存在多长时间
- Feature Flag需要多高的灵活性 (动态)

#### 1. Release Toggles
也称**发布开关**。
主要应用于使用trunk-based工作流的团队实践持续发布 (CD)。
在开发过程中，未完成的功能会直接提交到trunk/master分支中，使用release toggle来将这些功能关闭，确保这些未完成和未经测试的代码不会在线上被执行，直到功能完成之后再打开。

![](https://martinfowler.com/articles/feature-toggles/chart-1.png)

Release Toggles只能作为过渡性的解决方案，一般来说，一个Release Toggle只能存在一周或两周，之后就应该发布并删除开关。

#### 2. Experiment Toggles
**实验开关**一般用于实现A/B testing。
实现A/B testing之前需要先有用户分组功能；每个用户存在于一个固定的分组内，可以为每个组指定一个试验分支，通过与数据分析结合的方法来确定最终使用哪一个分支。

![](https://martinfowler.com/articles/feature-toggles/chart-2.png)

实验开关一般来说存在的时间会久一点，有时可能会长达数周。但是存在更长时间不一定更有用，因为很有可能新的代码引入使得之前的A/B对比结果不再准确。

#### 3. Ops Toggles
**运维开关**用于控制线上系统的运行时行为。
比如当我们引入一个新功能的时候，可能不清楚该功能对会系统整体性能产生什么样的影响，这时可以使用Ops Toggles来控制该新功能的关闭，或者是服务降级。

大部分Ops Toggles只会存在很短的时间，一旦功能在线上通过验证，这个开关就应该被删除。

![](https://martinfowler.com/articles/feature-toggles/chart-3.png)

但是也有一些Ops Toggle会始终存在于系统中，这类开关被称作"Kill Switches"，也就是当系统遇到问题时用于关闭部分功能，或者对某些功能做服务降级的开关。
比如说，当系统负载突然增加时，我们可能希望关闭首页上的推荐模块减轻服务器压力。这类开关控制的一般都是非关键业务关闭或降级。
这类长期存在的功能开关有时也被称作是 [Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)。

#### 4. Permissioning Toggles
**权限开关**可以用来将产品的某些功能只对指定的部分用户开放。
比如说，我们有一个高级功能只对付费用户开放；或者是有一个alpha功能只对我们的内部测试用户开放，以及一些beta功能只对内部测试用户和beta用户开放。

这种技术与金丝雀发布有些类似，区别在于，标准的金丝雀发布是对**随机的部分用户**开放功能，而权限开关是对**指定的用户组**开放功能。

![](https://martinfowler.com/articles/feature-toggles/chart-4.png)

权限开关一般来说会长期存在。另外权限开关在实现上需要更加的动态化，因为该功能是针对特定的用户组来做判断，所以每一个请求到达时都需要去检查；相对来说，这个功能的系统开销也就会更大。

#### 5. 如何管理不同类型的开关

##### A. static vs dynamic
![](https://martinfowler.com/articles/feature-toggles/chart-6.png)

动态类型的开关需要更多的toggle routers和toggle configuration来提供运行时信息及配置。

而简单的静态开关实现起来更容易，只需要toggle point自身的on/off状态就行。

比如实验开关，需要根据用户来做判断，还需要固定的分组算法，比如根据用户id来做分组等等。可能还需要从toggle configuration中读取分组规则设置，这些配置信息都需要作为参数传给分组算法。

##### B. Long-lived vs transient
![](https://martinfowler.com/articles/feature-toggles/chart-5.png)

对于短期开关，我们在实现上也可能会很简单，比如使用if/else语句来做判断就行。而对于长期存在的开关，一般来说我们会要求更好的设计，可能会用到某些设计模式来实现该功能。

**关于Feature Flag的实现技巧可以[参考Martin Fowler的文章](https://martinfowler.com/articles/feature-toggles.html)***


### 四、参考资料
1. What is a "feature flag" (很多地方引用了这个讨论话题)
[https://stackoverflow.com/questions/7707383/what-is-a-feature-flag](https://stackoverflow.com/questions/7707383/what-is-a-feature-flag)

2. 最早提到在实际项目中使用feature flag技术的文章
[http://code.flickr.net/2009/12/02/flipping-out/](http://code.flickr.net/2009/12/02/flipping-out/)

3. Martin Fowler关于Feature Flag应用的详细描述，包括Feature Flag的实现技巧
[https://martinfowler.com/articles/feature-toggles.html](https://martinfowler.com/articles/feature-toggles.html)

4. LaunchPad关于Feature Flag的介绍
[https://dev.launchpad.net/FeatureFlags](https://dev.launchpad.net/FeatureFlags)

5. Google Firebase的远程配置中心
[https://firebase.google.com/docs/remote-config/](https://firebase.google.com/docs/remote-config/)

6. LaunchDarkly的文档，Microsoft Visual Studio集成的Feature Flag功能来自Launch Darkly
[https://docs.launchdarkly.com/docs](https://docs.launchdarkly.com/docs) 

7. MSDN上的Feature Flag使用示例 (使用LaunchDarkly实现)
[https://blogs.msdn.microsoft.com/buckh/2016/09/30/controlling-exposure-through-feature-flags-in-vs-team-services/](https://blogs.msdn.microsoft.com/buckh/2016/09/30/controlling-exposure-through-feature-flags-in-vs-team-services/)

8. Feature Flag在Google Gmail开发过程中的使用
[https://gmail.googleblog.com/2011/12/developing-gmails-new-look.html](https://gmail.googleblog.com/2011/12/developing-gmails-new-look.html)

9. Reddit在开发过程中使用Feature Flag的案例
[https://github.com/reddit-archive/reddit/blob/master/r2/r2/config/feature/README.md](https://github.com/reddit-archive/reddit/blob/master/r2/r2/config/feature/README.md)

10. Feature Flag管理工具的商业化产品
[https://rollout.io/](https://rollout.io/)
[https://www.split.io/](https://www.split.io/)
[https://bullet-train.io/](https://bullet-train.io/)
[https://www.togglz.org/](https://www.togglz.org/) Feature Flag的开源实现 (Java)


