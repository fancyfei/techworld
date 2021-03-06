# Java异常体系与正确使用

程序处理过程中，遇到的各种出错，例如内存、硬盘、网络、代码错误等，这些非正常情况在Java中统一被认为是异常，有一整套异常机制来处理。

异常是相对于return的一种退出机制，可以由系统触发，也可以由程序通过throw语句触发，异常可以通过try/catch语句进行捕获并处理，如果没有捕获，则会导致程序退出并输出异常栈信息。

异常栈信息就是包括了从异常发生点到最上层调用者的轨迹，还包括行号，是分析异常最为重要的信息。

## 异常的类型体系

Java定义了非常多的异常类，它们以Throwable为根，分为两个方面，一个表示系统错误，一个表示应用程序错误。

异常有unchecked exception (未受检异常)，与checked exception (受检异常)。checked异常，Java会强制要求程序员进行处理，否则会有编译错误，而对于unchecked异常则没有这个要求。

![image-20210524151451798](img\java_exception_class.png)

- Throwable，是所有异常的基类，它有两个子类Error和Exception。

```java
public class Throwable {
    //它会将异常栈信息保存下来，这是我们能看到异常栈的关键。
    //它是性能差的原因，需要同步收集栈信息。
    Throwable fillInStackTrace();
    //异常可以形成一个异常链，上层的异常由底层异常触发，cause表示底层异常。
    //最多只能被调用一次。
    Throwable initCause(Throwable cause);
    //打印栈信息，有几个重载的方法。
    void printStackTrace();
    //获得异常信息
    String getMessage();
    Throwable getCause();
    StackTraceElement[] getStackTrace();
}
```

- Error，表示系统错误或资源耗尽，由Java系统自己使用，应用程序不应抛出和处理。它有两个常见错误子类，内存溢出错误(OutOfMemoryError)和栈溢出错误(StackOverflowError)。
- Exception表示应用程序错误，应用程序也可以直接继承它或其子类。
- RuntimeException，它比较特殊，它表示的实际含义是unchecked exception (未受检异常)。
- 自定义异常。

## 异常的代码块

```java
void method() throws zzzException {
    try {
        .... //异常发生点
    } catch (xxxException e) {
        ... //捕获异常结果后处理逻辑
        throw new yyyException();//再抛出处理后异常
    } catch (xxxxException e) {
        ...
    } finally {
        ...//不管有无异常都执行的逻辑
    }
}
```

- try块是需要捕获异常的代码块。
- catch块可以有多条，每条对应一个异常类型，异常处理机制根据抛出的异常类型按顺序匹配catch块，执行catch块内的代码，其他catch块就不执行了。

- 重新throw，catch块内处理完后，可以重新抛出异常。可以抛出新的异常或原来的异常。
- finally语句在catch后面，里面的代码不管有无异常发生，都会执行。一般用于释放资源。
- throws声明一个方法可能抛出异常。throws跟在方法的括号后面，可以声明多个异常，以逗号分隔。对于checked异常必须是要声明才能向上抛出的。
- try-with-resources 方式，JDK8后提供的一种声明了一种或多种资源的try语句。资源是要在程序用完了之后必须要关闭的对象，需要实现了java.lang.AutoCloseable接口。这个语句保证了每个声明了的资源在语句结束的时候都会被关闭。

## 正确的使用异常

使用异常重要的是一致性，一个项目中，应该对如何使用异常达成一致，按照约定使用即可。

异常大概可以分为三个来源：用户、程序员、第三方。用户是指用户的输入有问题，程序员是指编程错误，第三方泛指其他情况如I/O错误、网络、数据库、第三方服务等。每种异常都应该进行适当的处理。

处理的目标可以分为报告和恢复。报告时一种是报告给用户，另外一种是报告给系统如日志等，都需要表达准确与信息充分，以便让对方方便判断与处理。恢复是需要提供程序的容错机制。

在阿里的代码规范中，详细的提到了如何使用异常：

> 1，【强制】Java 类库中定义的可以通过预检查方式规避的 RuntimeException 异常不应该通过 catch 的方式来处理。
>
> 2，【强制】异常不要用来做流程控制，条件控制。
>
> 3，【强制】catch 时请分清稳定代码和非稳定代码，稳定代码指的是无论如何不会出错的代码。对于非稳定代码的 catch 尽可能进行区分异常类型，再做对应的异常处理。
>
> 4，【强制】捕获异常是为了处理它，不要捕获了却什么都不处理而抛弃之，如果不想处理它，请将该异常抛给它的调用者。最外层的业务使用者，必须处理异常，将其转化为用户可以理解的内容。
>
> 5，【强制】有 try 块放到了事务代码中，catch 异常后，如果需要回滚事务，一定要注意手动回滚事务。
>
> 6，【强制】finally 块必须对资源对象、流对象进行关闭，有异常也要做 try-catch。
>
> 7，【强制】不要在 finally 块中使用 return。因为覆盖try 块中的return。
>
> 8，【强制】捕获异常与抛异常，必须是完全匹配，或者捕获异常是抛异常的父类。

