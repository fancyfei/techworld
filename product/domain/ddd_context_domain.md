## 领域驱动设计，领域及限界上下文

复杂的业务一般总结来说是业务场景多、流程长、概念多，超过了单体架构下的个人所能理解或处理的范围。这时候解决办法就是大规模分工协作，共同设计解决方案。那么业务问题的边界问题、各部分沟通的一致性、系统架构快速响应业务变化就成了新的主要难题。

在DDD的解决方案中，首先要解决就是从业务中抽象出统一语言，划分问题域的边界，聚焦在核心域上，再迭代式的探索和发现模型。产出的成果是领域及子域，以及限界上下文。

## 战略设计阶段主要目标

战略设计阶段，以业务为驱动，为系统分析设计提供边界，以及优先级。

领域专家与开发团队一起，首先通过业务梳理对业务问题理解上达成一致，分析概念并消除二义性，划分概念边界，在边界内建立统一语言。

领域专家对业务问题具有最丰富的知识，帮助与指导开发团队进行分析设计。开发团队则要包括软件相关关键角色所有人，如客户、产品经理、UI、架构师、开发、测试、运维、DBA等人。

- **业务梳理**，整理与罗列业务所要解决的问题、业务场景和业务流程，进行沟通与交流，统一团队的认知，形成领域知识。最好是通过可视化的协作设计方式，直观的展示出各自的理解。业务抽象的可视化工具有：四色建模、事件风暴等。
- **识别限界上下文**，在业务梳理和抽象完成后，将领域名词和外部相关的系统，根据概念的相关性进行分类，划分出概念边界，即限界上下文。并对限界上下文间概念的依赖关系进一步分析，实现概念边界上的内聚和解耦。得到的产物是限界上下文与映射图，也是业务问题最小粒度的划分及其之间关系。
- **识别子域**，根据限界上下文，按照业务问题独立性划分不同的问题子域。一般子域类型有核心域、通用子域、支撑子域。每个子域都是一个业务问题解决域，是对业务问题更大粒度的划分，实现的是业务边界上的内聚和解耦。

## 限界上下文

> 限界上下文（Bounded Context）定义了每个模型的应用范围，在每个Bounded Context中确保领域模型的一致性。每一个上下文内部紧密组织，职责明确，具有较高的内聚性。

领域包含了问题域和解决方案域，限界上下文就是软件对于问题域的一个特定的、有限的解决方案。根据业务梳理的通用语言中的概念（需求中的术语、领域名词），按领域划分限界上下文。

划分的策略是从概念之间找到他们的联系，分析他们之间互动关系，将紧密耦合的概念放一起形成限界上下文，再描述此限界上下文的职责，如果不清楚、准确、简洁、完整，则重新分析归纳，直到得到合适的限界上下文。

限界上下文的重点：业务问题上的最小粒度、概念无二义性。

> 上下文映射图（Context Map）是表示各个系统之间关系的总体视图，主要表明相互集成关系。

上下文映射图用于高层次的架构分析，只从解决方案空间的角度看待各限界上下文的关系，并非企业架构与系统拓扑图。需要随着业务变化而更新。限界上下文之间的组织模式和集成模式有：

- 合作关系，两个限界上下文的共同满足两个领域的需求。
- 共享内核，对模型和代码都共享。
- 客户方-供应方开发，是一种上下游关系，上游满足下游的需求。
- 跟随者，也是上下游关系，上游对下游做出承诺，下游跟随实现。
- 防腐层，两个设计良好的限界上下文，通过委派一个中间层提供功能。
- 开放主机服务，开放协议，供所有与之交互的限界上下文各自集成。
- 发布语言，开放主机服务一起发布的共享语言来集成与交流。
- 各行其道，相互独立，互不影响。
- 大泥球，混杂模糊的边界。

通过上下文映射关系，我们明确的限制了限界上下文的耦合性。

## 子域划分与识别

通过对于子域进行识别、划分和类型标注，团队能够实现软件架构在业务边界上的内聚和解耦。

- 核心域，整个业务运作的基础，代表这个领域内的核心竞争力与整个业务基石。

- 通用子域，非常常见并有现成解决方案，可通过简单实现或购买到。

- 支撑子域，支撑核心域的运作，有个性化的需求，需要定制的。

领域与子域是包含关系，多个子域组合成更大的域。最大的域则是领域。

核心域是领域中最重要的部分，需要投入最优资源，做最严谨的设计来保障。开发者和领域专家花费最多的精力都是在它上面，一个业务系统一般有且只有一个核心域。对核心域的精炼工作需要花费很大的时间与精力，确定完核心域后再围绕它展开的对通用子域、支撑子域进行分析与精炼。

核心域精炼过程是个持续的过程。有以下几种方式：

- 领域愿景说明，简短描述核心域，将要创建的价值等。这就是个指南，供所有人参考与保持方向。
- 突出核心，突出核心域中的领域模型有两种方式，精炼文档与标明核心。
- 内聚机制。
- 分离的核心，对模型重构，将非核心模型移出核心域，将核心模型移动到核心域。
- 抽象核心。

