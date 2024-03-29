# 设计原则

一个好的软件设计要解决的核心问题是，如何同时提高一个软件系统的可维护性和可复用性。

可维护性就是能够允许新的设计要求以较为容易和平稳的方式加入到已有的系统中。可维护性较低的原因通常有：

- 过于僵硬，新功能加入会涉及很多其他模块。小改动都有大范围波及。
- 过于脆弱，僵硬的同时，一个小改动引起其他模块的故障，且无法预测影响范围。
- 复用率低，过多的依赖让一个功能模块无法被其他模块复用。
- 黏度过高，修改总是以破坏原始意图及框架，这种权宜之计最终会导致错误的维护方案，长期来看则无法维护。

一个好的系统设计应该有如下的性质：

- 可扩展性，新功能很容易加入。
- 灵活性，对系统的修改能平稳的进行，影响只在一个小的范围。
- 可插入性，很容易将一个类抽出去，或者同样接口的类加入进来。

软件复用的好处是：

- 较高的生产效率。
- 较高的软件质量。
- 适度复用可以改善系统的可维护性。

软件复用的形式：

- 代码片段复用、算法的复用、数据结构的复用，这些复用核心还是代码层面的复用。
- 框架、模块、商业逻辑的复用，在代码复用的基础上，上升到宏观层面的复用。这些复用都是是设计原则和设计模式为基础的。

设计原则首先都是复用的原则，可以提高系统的复用性，同时提高可维护性。

## 设计模式遵循的原则：

| 设计原则     | 原则的内容                                         | 目的                                         |
| ------------ | -------------------------------------------------- | -------------------------------------------- |
| 开闭原则     | 对扩展开放，对修改关闭                             | 降低维护风险，提高维护性                     |
| 依赖倒置原则 | 高层次模块不应该依赖低层次模块，应当依赖于抽象接口 | 降低耦合，提高稳定性、可读性、可维护性       |
| 单一职责原则 | 每个类都应该有一个单一的功能                       | 降低类的复杂度，提高可读性、可维护性、扩展性 |
| 接口隔离原则 | 一个接口只干一件事，拒绝庞大接口                   | 功能解耦，高聚合、低耦合                     |
| 迪米特法则   | 尽量不要过多的与其他类互相作用，减少类之间的耦合度 | 降低类之间的耦合，减少代码臃肿               |
| 里氏替换原则 | 使用父类的地方可以任意使用其子类                   | 增强程序的健壮性                             |
| 合成复用原则 | 复用尽量使用组合或者聚合，少使用继承               | 降低代码耦合                                 |

## 原则对设计目标的支持

这些从许多设计方案中总结出的指导性原则，都是为了保证三个设计目标的达成。

- 可扩展性的保证：开闭原则、里氏替换原则、依赖倒置原则、合成复用原则。
- 灵活性的保证：开闭原则、迪米特法则、接口隔离原则。
- 可插入性的保证：开闭原则、里氏替换原则、合成复用原则、依赖倒置原则。

在以上原则中，最终目的是对扩展开放、对修改关闭，即开闭原则，其他的原则都是为了达到这个目的。

