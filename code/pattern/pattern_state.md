# 状态模式

状态（State）模式，是一种对象行为型模式。把复杂的“判断逻辑”提取到不同的状态对象中，让你能在一个对象的内部状态变化时改变其行为。

其主要思想是用有限状态机替换swith...case...和if...else...。所面向的业务对象在任意时刻仅可处于几种有限的状态中，不同的状态有不同的行为，有预定义各状态的切换规则。

使用状态模式主要解决两种问题：

- 状态不同，行为不同。状态抽象出来，把不同的行为逻辑分离。
- 行为与状态之间有很多条件判断。用状态变化驱动行为的发生，状态变更的判断分别由不同具体状态类封装。

在工作流中使用的就是这个思想，一个流程有很多环节，用状态驱动流程的节点变化。

## 状态模式的实现

状态模式主要是要抽象出来状态，并找出状态对应的行为。主要角色有：

- 环境类（Context）角色：也称为上下文，它定义了客户端需要的接口，实际调用状态接口；内部维护一个当前状态，并负责具体状态的切换。
- 抽象状态（State）角色：定义状态的行为接口，用以封装环境对象中的特定状态所对应的行为，可以有一个或多个行为。
- 具体状态（Concrete State）角色：实现抽象状态所对应的行为，并且在需要的情况下进行状态切换。它有上下文对象的反向引用，可从中获取信息，触发状态转移。

- 客户类（Client）角色：通过上下文对象处理请求和主动变更状态。

类图如下：

![pattern_state](pattern_state.png)

代码如下：

```java
//环境类
@Data
class Context {
    private State state;
    //定义环境类的初始状态
    public Context(State state) {
        this.state = state;
        this.state.setContext(this);
    }
    //对请求做处理
    public void doThis() {
        state.doThis(this);
    }
    // state set get
}
//抽象状态类
@Data
abstract class State {
    private Context context;
    public abstract void doThis();
}
//具体状态A、B...类
class ConcreteStateA extends State {
    public void doThis(Context context) {
        // 处理业务，按规则是否改变状态
        context.setState(new ConcreteStateB());
    }
}
//客户端使用时
Context context = new Context();
context.doThis();
```

## 状态模式的应用

状态模式的主要目的是将不同状态的行为分离开，将状态转换显式的暴露出来。在这个目标下，可以对状态类进行简化：

- 对状态本身可简化：对于状态之间的简单的转换可以不在状态内部进行，可减少不必要的深度。
- 状态类中组合环境类关系简化：可以直接在状态的行为方法中参数传递环境类。

状态模式的结构与实现都较为复杂，会增加很多类，使用时需要注意适当的规模和结构。

在状态转换不明显的场景下，可以向策略模式靠近，也能做到不同状态不同行为分离的目的。

