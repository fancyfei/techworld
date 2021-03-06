# JVM的方法调用

方法调用的主要任务就是确定被调用方法的版本（即调用哪一个方法），该过程不涉及方法具体的运行过程。又分静态解析调用和分派调用。

在Class文件中，所有方法调用中的目标方法都是常量池中的符号引用。

- 静态解析调用：在类加载的解析阶段会将一部分符号引用转为直接引用，也就是在编译阶段就能够确定唯一的目标方法。主要包含以下几种方法:静态方法、私有方法、实例构造器、父类方法。

- 分派调用：与重载和重写特性相关的等，也分编译阶段确定的静态分派和运行时才确定的动态分派。

Java虚拟机一共提供了 5 条方法调用字节码指令，分别是：

- invokestatic：调用静态方法。
- invokespecial：调用实例构造器<init>方法、私有方法和父类方法。
- invokevirtual：调用所有的虚方法。
- invokeinterface：调用接口方法，会在运行时确定一个实现该接口的对象。
- invokedynamic：先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

分派调用则即可能是静态，也可能是动态的：

- 静态分派：重载，参数静态类型；依赖静态类型来定位方法执行版本。
- 动态分派：重写Override，参数动态类型；在程序运行时根据变量的实际类型来分派方法的执行版本。

分派调用根据分派宗量可以分为单分派和多分派（方法的接收者和方法的参数统称为方法的宗量）：

- 单分派：根据一个宗量就可以知道调用目标。
- 多分派：根据多个宗量才能确定调用目标。

JDK1.7以前版本的java语言是一门静态多分派、动态单分派的语言。 

## invokedynamic，动态方法调用新指令

在使用了invokedynamic的类的常量池中，会有一个特殊的常量，invokedynamic操作码正是通过它来实现这种灵活性的。这个常量包含了动态方法调用所需的额外信息，它又被称为引导方法（Bootstrap Method，BSM）。每个invokedynamic的调用点（call site）都对应着一个BSM的常量池项。

invokedynamic指令的调用点在类加载时是“未链接的（unlaced）”。调用BSM后才能确定具体要调用的方法，返回的这个CallSite对象会被关联到调用点上。

方法句柄（Method Handle）这套新的反射API，真正全面支持了动态方法调用。

