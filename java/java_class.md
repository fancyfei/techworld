# Java的数据类型：类
Java数据类型有8个基本类型（字符类型char，布尔类型boolean以及数值类型byte、short、int、long、float、double）。
还有3种引用类型（类、接口、数组）。

在Java类的体系中还包括了几个特殊作用的类：抽象类，内部类，枚举。
## 类
类是 Java 中的一种重要的引用数据类型，也是组成 Java 程序的基本要素，因为所有的 Java 程序都是基于类的。
- 使用 class 关键字定义
- 公共的类名必须与文件名同名，必须是public的。
- 入口是public static void main(String[] args)

## 内部类
内部类（ Inner Class ）就是定义在另外一个类里面的类。与之对应，对外可见的那个类被称为外部类。
内部类是一种更好的封装，把内部类隐藏在外部类之内，同一个包中的其他类都不能访问该类。
同时内部类可以实现间接的多继承。
- 内部类的方法可以直接访问外部类的所有数据，包括私有的数据。
- 内部类所实现的功能使用外部类同样可以实现，只是有时使用内部类更方便。
- 缺点是使得类结构变得复杂。
- 根据用法再细分为：成员内部类、局部内部类、静态内部类、匿名内部类等。

## 成员内部类
作为外部类的一个成员存在，如同类的属性、方法，是外部类的重要组织部分，自己不会单独存在。
应用场合：每一个外部类对象都需要一个内部类的实例。
- 成员内部类持有外部类的引用。
- 内部类通过this可以访问外部类的属性与方法。
- 成员内部类中不能定义static属性和方法
- 外部类先创建内部类的对象，然后通过内部类的对象来才能访问其属性和方法。

## 局部内部类
局部内部类也叫区域内嵌类，局部内部类与成员内部类类似，不过，是定义在一个方法中的内嵌类，只能被此方法使用。
- 可持有外部类对象引用
- 不能使用private，protected，public修饰符
- 不能包含静态成员

## 静态内部类
成员内部类如果使用static声明，则此内部类就称为静态内部类。可以通过“外部类.内部类”来访问。
静态内部类和静态属性类似，可以访问外部的静态变量，通过外部类的实例访问其属性与方法。

## 匿名内部类
匿名内部类也就是没有名字的内部类，只能使用一次，即用即创建并实例化，只关注其业务逻辑，所以使用也比较广泛。
- 使用new创建 ，没有具体位置。
- 创建的匿名类，立刻就实例化。
- 可持有外部类对象引用。
- 内嵌匿名类编译后生成的.class文件的命名方式是”外部类名称$编号.class”，编号为1，2，3…n，编号为x的文件对应的就是第x个匿名类。
## 枚举类
是一种有特殊约束的类，具有简洁性、安全性以及便捷性。
由关键字enum创建。
枚举类型并编译后，编译器会为我们生成一个相关的类，这个类继承了Java API中的java.lang.Enum类。

## 抽象类
抽象类前使用abstract关键字修饰,则该类为抽象类。
约定子类需要实现哪些方法，并能提供一些子类共有的属性与方法。
从多个具有相同特征的类中抽象出一个抽象类，其最重要的作用是作为子类的模板,从而避免了子类设计的随意性。
- 限制规定子类必须实现某些方法,但不关注实现细节。
- 抽象类不能直接创建,可以定义引用变量。
## 接口
接口可以理解为一种特殊的类,由全局常量和公共的抽象方法所组成。
- 使用interface关键字定义接口。方法默认是abstract、public的。
- 接口中的属性是常量。
- 接口可以多继承。并且无法实例化。
- 接口中的方法只能是抽象方法。在JDK8以后，可以有默认实现default。
- 一个类可以实现一个或多个接口,实现接口使用implements关键字。
- 有一种没有任何方法的“标记接口”：作用就是给某个对象打个标，处理此对象时使对象附加某个或某些特权。