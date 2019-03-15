### 发布新版本的方法：Feature Flags还是Rings

*使用rings形式的逐级发布，同时为每个新的功能添加feature flag*

![](https://opensource.com/sites/default/files/styles/image-full-size/public/lead-images/biz_feature_freeze2_0.jpg?itok=ZBAz0b6b)

**DevOps的目标是快速发布**。
通过持续的快速发布，以及实时的用户反馈，我们能够不断对产品做修正，保留那些对用户体验和留存有提升的功能，而那些不能带来好的结果甚至是有负面影响的功能，则立即关闭。

**DevOps的实践就是渐进式迭代**。
基于feature flag和ring形式的逐级发布，能够让我们把变更只对特定的用户群开放，同时观察并验证结果是否符合预期，然后才扩大到所有用户。

你可能会问，到底是用ring形式的逐级发布好，还是用feature flag好，或者两种都支持？让我们来探索一下这些策略。

### Rings
基于ring形式的逐级发布方法最早由Jez Humble提出，[Continous Delivery](https://www.continuousdelivery.com/)，同时也用到了**金丝雀发布**。
Rings将变更的影响只限定在一部分用户范围内，同时逐步地放开并观察变化所造成的影响。
通过ring发布，我们能够评估影响，或者形象的比喻叫做限制爆炸的影响范围。通过观察、测试、诊断，以及最重要的，用户反馈，ring发布使得我们能够渐进式的发布新版本，并且线上会并行跑着多个版本。
你可以实时获得用户对新版本的反馈，并且不用担心影响到所有的用户；直到确定新版本已经没有问题之后，丢弃老版本，全部使用新版本。

下图展示了**ring发布的执行流程**：

![](https://opensource.com/sites/default/files/images/life-uploads/ring-based-deployment.png)

当开发人员将pull request合并到master分支后:
1. CI开始构建该应用，执行单元测试，以及触发在金丝雀测试环境的发布。当金丝雀测试环境发布通过之后，自动发布到QA测试环境
2. QA确认没有问题后，自动发布到早期测试环境；同样的，在这个环境下也需要确认发布结果是否符合预期
3. 最后将发布推到所有用户圈。具体每个ring的名字以及用户数量根据具体情况来设计，但是需要保证的是，每个ring的运行环境都是完全相同的

### Feature flags
Feature Flag因为[Martin Fowler](https://martinfowler.com/bliki/FeatureToggle.html)的推广而被人所熟知。
Flags将发布与功能解耦，可以在运行时为每个用户指定功能的开启或关闭，并且支持预测驱动的开发模式。
通过将feature flag与监控相结合，还能够知道新功能是否对提升用户满意度、留存等有效果。
当然也可以用feature flag来实现某个功能的紧急回退，或者在某些地区关闭一些特定功能，以及在需要的时候开启某项监控，等等。

![](https://opensource.com/sites/default/files/images/life-uploads/ff-switch.png)

##### Feature flag的典型实现:
1. 在管理后台上定义flag，以及能够修改flag的值
2. 提供接口用于实时查询flag的值
3. 在代码中使用if-else结构来进入不同的分支流程

![](https://opensource.com/sites/default/files/images/life-uploads/feature-flags_0.png)

不论是feature flag还是基于ring的逐级发布都是非常有价值的；不论你是在写一个开源的插件，还是在有65000人的大型组织内实践DevOps。

### 回到我们最初的问题：选择feature flag还是rings，或者两者都要？

我们同时使用rings和feature flag来发布新功能，小到bug修复大到新功能发布，用的都是这种方法。我们的产品服务于超过65000个开发人员，每次发布时小范围用户圈则有成百上千人。

Feature flag允许我们在发布的时候渐进式开放某些功能，执行A/B testing，以及在产品层面的实验。因为用户在云平台上使用我们的产品，所以我们能够快速从用户那里获得持续的反馈；使用feature flag我们也能够调整某些功能。

对于要发布给用户的插件类产品来说，我们主要采用基于ring形式的渐进式发布。先从早期用户开始，再到最终用户，扩展的方法可以参见下表。我们持续的用feature flag来添加一些功能，验证并搜集用户反馈，然后再不断的调整。

[Facebook的发布系统](https://code.fb.com/developer-tools/rapid-release-at-massive-scale/)也展示了同样的策略：先将新功能发布给1%的用户，然后20%，50%，最后才是所有用户。

对于实践DevOps的渐进式发布来说，feature flag和ring都能实现，它们是共生的。两者的目标也是一致的：当你想要将某个功能发布给用户时，你希望能够采用渐进式的策略，并且在出现问题时能够快速回滚。我的建议是两种方案都尝试。从发布到金丝雀测试环境开始，渐进式的发布新功能，然后使用feature flag来调整产品功能。

![](https://opensource.com/sites/default/files/images/life-uploads/trolley.png)
在我看来，基于ring的发布就是一辆手推车，逐步向前推进。

![](https://opensource.com/sites/default/files/images/life-uploads/screwdriver.png)
而feature flag则是一把螺丝刀，可以用来对产品不断做调整。

**Happy ringing and flagging!**

### 基于我们的产品使用ring和feature flag的实践做的对比


|  | Deployment Ring | Feature Flag |
| --- | --- | --- |
| 支持渐进式发布 | 是 | 是 |
| 支持A/B testing | ring内的所有用户 | 所有用户或者选择的部分用户 |
| 维护开销 | 发布环境维护 | Feature flag数据库，代码维护 |
| 主要用途 | 控制变更的影响面 | 开启或关闭某项功能 |
| 影响面 - Canaries | 0 - 9金丝雀用户 | 0, all, 或选择的部分金丝雀用户 |
| 影响面 - Early Adopters | 10 - 100早期测试用户 | 0, all, 或选择的部分早期测试用户 |
| 影响面 - Users | 10000+用户 | 0, all, 或指定的用户 |


**原文链接：**[https://opensource.com/article/18/2/feature-flags-ring-deployment-model](https://opensource.com/article/18/2/feature-flags-ring-deployment-model)