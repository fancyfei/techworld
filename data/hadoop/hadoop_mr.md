# MapReduce: 分布式计算框架

MapReduce系统是一个分布式计算框架,可以理解为是一个jar包或一个程序，这个程序要运行在Yarn上面。主要任务就是利用廉价的计算机对海量的数据进行分解处理。它的计算方式是指定一个Map函数把一组键值对映射成一组新的键值对，再用Reduce函数并发的进行归集。详细的过程是先读取文件数据，然后进行Map处理，接着Reduce处理，最后把处理结果写到文件中。

![mapreduce_process](img/hadoop_mapreduce_process.png)

## 框架特点

- 易于编程：有简单充分的接口，可以很容易完成对一个复杂的分布式程序的编写。
- 易扩展性：增加机器即可增加计算能力。
- 高容错性：节点异常，计算会转移到健康的节点，不影响全局结果。
- 海量数据的离线处理：只要有足够的计算能力，PB级的数据也可快速处理。适合批量计算。

## 处理过程

Map过程：

1. 输入数据，可分解为多个块，以便并行处理。
2. 启动多个Map线程（MapTask），进行数据的处理（映射），然后放在缓冲区。
3. 进行分区、排序等操作，写硬盘。
4. 数据处理完成后，对硬盘上的所有文件进行合并排序。

Reduce过程：

1. 启动多个Reduce线程（ReduceTask），接收MapTask的文件，放内存，超大则合并后存硬盘
2. 对硬盘上文件做排序合并等处理。
3. 输入到reduce函数，进入数据的处理（归集）。
4. 输出结果到硬盘。

用户可以只需要实现 Mapper和Reducer，其他的使用默认实现就可以了。同时InputFormat、Partitioner和OutputFormat等也可以用户来实现。

MR过程的Task由YARN管理，它是一个纯粹的资源管理调度框架，资源管理交给了全局的资源管理器ResourceManager，任务方面的调度和监控交给了应用相关的ApplicationMaster。

## 常用的类

通过InputFormat决定读取的数据的类型，然后拆分成一个个InputSplit，每个InputSplit对应一个Map处理，RecordReader读取InputSplit的内容给Mapper。Partitioner对数据按照key进行分区，再给Reducer处理，通过OutputFormat输出到硬盘。整个过程中，都是RecordWriter收集中间或结果数据，再输出到硬盘。

