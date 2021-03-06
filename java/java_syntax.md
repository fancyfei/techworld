# Java语法的基础
基本注意点：
- 大小写敏感，类名一般大写，方法名一般小写。
- 源文件名必须和类名相同。
- 所有的程序入口是public static void main(String []args)。
- javac/java 编译/执行

> 计算机由CPU、内存、硬盘和输入输出设备组成，所有的编程语言都会提供大量相应API，需要非常熟练。

## 基本数据类型
* 整数类型：有四种整型 byte/short/int/long，分别有不同的取值范围，8位/16位/32位/64位
* 小数类型 ：有两种类型 float/double，有不同的取值范围和精度，32位/64位
* 字符类型：char，表示单个字符，16 位 Unicode 字符

```
类型和对应的取值范围
byte: -2^7 ~ 2^7-1
short: -2^15 ~ 2^15-1
int: -2^31 ~ 2^31-1
long: -2^63 ~ 2^63-1
float: 1.4E-45 ~ 3.4E+38 和 -3.4E+38 ~ -1.4E-45
double: 4.9E-324 ~ 1.7E+308 和 -1.7E+308 ~ -4.9E-324
```
> Java语言提供了八种基本类型。六种数字类型（四个整数型，两个浮点型），一种字符类型，还有一种布尔型。
>
> 小数计算的结果不精确，如果需要精确的计算，一种是使用BigDecimal类，另外就是先转为整型计算出结果后再转回小数。
>
> Java除了基本类型还有一个【引用类型】，分别有指向类、接口、数组、null等类型。
> 
> 每种基本类型都对应一个包装类。
> 
> 引用类型不压缩占8位，压缩4位。
> 
> 对象占内存情况：普通对象头，不压缩占16位，压缩12位；数组对象头不压缩占24位，压缩16位；

## 基本运算
- 算术运算：加减乘除，取模(%)，自增/自减(++ --)
- 比较运算：大于(>)，大于等于(>=)，小于(<)，小于等于(<=)，等于(==)，不等于(!=)，结果是布尔类型的值
- 逻辑运算：与(&)，或(|)，非(!)，异或(^)，短路与(&&)，短路或 (||)，结果生成一个布尔值
> 自增/自减是"快捷"操作，放变量前则先操作后使用，放变量之后则先使用后操作。
> 
> 短路是前面部分可以推送结果，后面部分则被忽略。
>
> 运算的优先级问题搞不明白的话，最好使用括号()来表达我们想要的顺序。

## 赋值操作
- 声明变量就是在内存分配了一块区域，赋值就是向这块区域设置具体的内容
- Java中有两种不同的赋值，基本类型、数组或对象的赋值
- null是一种特殊的类型（type），可以将它赋给任何引用类型变量，表示这个变量不引用任何东西；
- 还有一种特殊的class literal，用于表示类型本身
- Java是强类型语言，除了基本类型外，不同的类型人变量是不能赋值
- 基本数据类型之间转换是小转换到大，可以自动完成（隐式转换），而从大到小，必须强制转换（包括short和char之间）；
大小是：byte <（short=char）< int < long < float < double
- 基础类型强制转换，也被称作缩小转换，整数被取余，浮点转整形会截尾。

> 基本类型赋值：直接将内容设置到指定的区域，对变量存取的都是真实的内容
>
> 数组或对象赋值：分内容存储区域和位置存储区域，对变量存取的是位置（指针），如需存取内容则需要“.”(对象)或“[]”(数组)

## 过程控制（条件与循环）
- 条件：if/else，switch，三元运算符，其中switch的表达式值的数据类型只能是 byte, short, int, char, 枚举, 和String
- 循环：while, do/while, for, foreach，循环的控制是break和continue，