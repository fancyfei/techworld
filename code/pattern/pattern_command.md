# 命令模式

命令（Command）模式，是一种对象行为型模式，将一个请求封装为一个对象，调用者发送，实现者根据命令对象做对应的响应，调用与响应分离。

核心逻辑是：把一组业务操作与调用方分开，通过抽象出的命令衔接。

封装请求的好处在于，将执行方法参数化，并且对命令本身可以做更多的控制，如延迟处理、撤销等。也便于增加新的命令扩展业务逻辑。

JDK中的Runnable是一个典型命令模式，Runnable担当命令的角色，Thread充当的是调用者，start方法就是其执行方法。Runnable中的run执行的就是具体的业务了，可以组合业务对象做接收者。

## 命令模式的实现

命令模式中最主要的事是命令的抽象，主要的角色如下：

- 抽象命令类（Command）角色：声明执行命令的接口，通常仅一个。
- 具体命令类（Concrete Command）角色：是抽象命令类的具体实现类，自身并不完成工作，它拥有接收者对象，把命令要执行的操作委派给接收者完成。
- 实现者/接收者（Receiver）角色：真正执行命令功能的相关操作。
- 调用者/请求者（Invoker）角色：是请求的发送者，它通常拥有很多的命令对象，并通过访问命令对象来执行相关请求，它不直接访问接收者，也不创建命令。
- 客户端 （Client）角色：创建命令、创建调用者。其中命令中封装了接收者，命令关联给调用者。

类图如下：

![pattern_command](pattern_command.png)

代码如下：

```java
// 调用者
@Data
class Invoker {
    private Command command;
    // 调用命令
    public void executeCommand() {
        command.execute();
    }
    // command set get
}
//抽象命令
interface Command {
    public abstract void execute();
}
//具体命令
class ConcreteCommand implements Command {
    private Receiver receiver;
    //关联接收者
    public ConcreteCommand() {
        receiver = new Receiver();
    }
    //让接收者执行真正的操作
    public void execute() {
        receiver.operation();
    }
}
//接收者
class Receiver {
    public void operation() {
        //命令真正执行的具体业务
    }
}
//客户端执行过程
Command cmd = new ConcreteCommand();
//也可在这儿给Command关联接收者
Invoker ir = new Invoker(cmd);
ir.executeCommand();
```

## 命令模式的应用

从一组相似的业务中抽象出命令，才具备使用命令模式的前提。也可以将多个命令组合成一组宏命令。

重点要区分于策略模式，他们都将行为参数化，但是命令模式主要包装操作及参数，策略模式主要是方便切换算法。

常见的命令模式的场景是：想让操作参数化的、延迟执行的、部分异步的和远程的操作、希望有回滚的等等。
