### 构建Pintrest的A/B测试平台

作为一家数据驱动的公司，Pintrest重度依赖实验来指导产品开发。
在任何时候，Pintrest的线上都有超过1000个试验在运行中，并且每天都有新的试验添加进来。

因为每天都要操作实验功能，搜集并验证试验数据，所以我们需要一个简单可靠的管理平台。同时为了减少操作过程中出错的可能性，我们引入了一个轻量级的配置UI，一套QA工作流和一组简单的API接口，用来支持跨平台的A/B测试。

##### 在构建平台的过程中，我们优先考虑如下的需求：
- 实时的配置变更：我们需要能够立即关闭或开启某个试验，并且不需要走代码更新流程；这在某个试验导致了线上事故的时候尤其重要
- 简单的配置过程：添加一个试验只需要配置一个开关，并且操作者几乎不可能会犯错
- 对用户透明：操作者只需要学习一次，就可以在所有平台上用同样的方法来创建试验
- 分析：试验的目的是根据试验结果做更好的决策，我们需要一个好用的数据分析控制台
- 可扩展性：整个系统都需要可扩展，包括业务服务和离线数据处理部分


##### 简单的处理流程：
Pinterest的试验遵循如下原则：
- 每个试验都要有默认配置，需要有对试验数据的预期，验证该预期的方法，以及完整的描述文档
- 每个试验都应该有确定的业务类型、适用组和禁用组，使用过滤器来限定目标用户
- 根据试验的结果，可以选择把功能代码应用到全部用户，或者是回滚该功能；每个试验在完成后都需要有总结文档

在早期的框架里，上述过程都完全通过代码来实现。
我们希望在新的框架上能够通过UI来编辑这些功能，并且能够获得实时反馈和正确性校验。
并且，试验规则的修改通过配置中心来完成，以实现配置与代码的解耦。

![](https://cdn-images-1.medium.com/max/800/0*FAUJkRfh7HYfN60Q.png)

一些常见的错误类型，比如语法错误、分组不平衡、测试组用户存在重叠、或违反了试验过程等，都能在操作的同时进行校验并显示出错误原因。
我们甚至还开发了实时的输入提示用来减少用户的输入，现在我们创建一个新的试验只需要很少的几次点击就可以完成。

![](https://cdn-images-1.medium.com/max/800/0*gMnKpRgXpql-p6br.png)
![](https://cdn-images-1.medium.com/max/800/0*7rguszFUlbKob8ls.png)


试验配置序列化之后会立即同步到所有服务节点，整个过程只需要几秒钟就能完成。
这是一个配置项经序列化后的示例：

```
{
"holiday_special": { 
        "group_ranges": { 
            "enabled": {"percent": 5.0, "ranges": [0, 49]},
            "hold_out": {"percent": 5.0, "ranges": [50, 99]}
        },
        "key": "holiday_special", 
        "global filter": user_country(‘US’),
        "overwrite_filter": {"enabled": is_employee()},
        "unauth_exp": 0,
        "version": 1
    } 
}
```

将配置与代码分离带来的好处是可以立即更新试验配置。
比如想要增加一个试验组的人数，不需要更新代码，直接就能发布修改。这使得试验的修改从产品发布周期中解放出来，加速了试验的迭代过程。这个功能在需要做紧急更新时尤其有用。

##### 质量保证

一个最简单的试验也有可能会影响到上百万的业务数据，所以我们对试验操作的标准要求非常高，并且有很严格的质量保证工具来支持。
我们的管理后台有一个试验审核工具，每一次试验的变更都需要进行审核。下图显示了审核流程中某个试验的配置修改，审核人员会收到邮件通知，然后通过UI来执行审核操作。

![](https://cdn-images-1.medium.com/max/800/0*LueBfzQndxSMwzVT.png)

大部分的试验都会有一个跨部门的小组来推进，小组成员包括平台开发人员、产品经理和数据分析师。
产品经理做出的每一次变更都需要由小组来共同审核，审核的关键点包括检查计划、预期、关键结果、触发逻辑、过滤器设置、分组校验以及文档。
这个流程在我们的web app上会强制执行。

如果某个试验有相应的代码修改，比如修改了决策逻辑里的控制组信息。我们要求试验人员将代码Pull Request添加到试验平台上，这样方便分析人员进行跟踪，以及在需要的时候进行调试。同时，我们也会把管理平台上的修改记录以评论的形式添加到Pull Request上。

![](https://cdn-images-1.medium.com/max/800/0*s2adMzjd2sXmi-Cg.png)

在UI上还有一个功能是一键复制试验，这主要针对测试人员的使用场景。
复制出来的试验仅供测试使用，在这里做的修改不会影响到原始的试验配置；测试人员也可以一键将试验发布到正式环境。

![](https://cdn-images-1.medium.com/max/800/0*HtjPwdVsz8U1LWKQ.png)

#### API
应用程序代码通过试验API来获取配置，并与业务代码进行关联，两个关键方法如下：
```
def get_group(self, experiment_name)
def activate_experiment(self, experiment_name)
```

**get_group**方法返回要使用的的试验组
**activate_experiment**记录该试验的使用行为

这两个方法基本上覆盖了A/B测试的主要用例，使用方法示例：

```
# Get the experiment group given experiment name and gatekeeper object, without actually triggering the experiment.
group = gk.get_group("example_experiment_name") 

# Activate/trigger experiment. It will return experiment group if any.
group = gk.activate_experiment("example_experiment_name") 

# Application code showing treatment based on group.
if group in ['enabled', 'employees']:  
    # behavior for enabled group  
    pass
else:  
    # behavior for control group  
    pass
```

gatekeeper对象封装了实验需要的user/session/meta信息。
除了上面展示的Python库以外，我们同时也提供了Scala、Java、Javascript、Android和iOS库。

#### 设计和架构

![](https://cdn-images-1.medium.com/max/800/0*tKJd1kXZ38PY1yAw.png)

实验平台逻辑上划分为三个模块：
- 配置系统
- API
- 数据分析

配置系统保存通过web UI修改的配置内容到我们的实验数据库中，这些信息一般来说会在一分钟内同步到所有的服务器节点。

实验客户端获取配置信息，通过调用API来决定实验逻辑，比如实验类型和分组信息。

客户端生成的日志被发送到消息队列，在这里分析管线会创建实验报告。
试验人员需要先定义数据分析指标，处理程序会实时分析日志并生成报告，之后报告被发送到控制台。



[原文地址](https://medium.com/@Pinterest_Engineering/building-pinterests-a-b-testing-platform-ab4934ace9f4)