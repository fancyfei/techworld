# MapReduce算法与编程模式

map（映射）和reduce(归约，化简）是数学上两个很基础的概念。前者将集合转换到另一个集合，后者将特定集合转化出结果。

它们在各类的函数编程语言里是高阶函数，map函数将遍历一个list，并把每个成员上交给另外一个函数处理，得到一个新的list，交给reduce函数对这新的list进行最后归集计算。

## MR计算框架

Hadoop中的MapReduce实际上就是在两个概念组合并加上并行化计算的编程模型。是一种分而治之的思想，将复杂的海量的问题层层分解为最简单子问题，再并行处理每个子问题，直到最后的问题都解决，再将子问题的结果层层合并归集得到最终结果。

MapReduce的框架中，统一处理了大量的存储、分发、错误处理、并行调度等等逻辑，开发人员只需要关注在业务处理环节上，即map、reduce、或shuffle几个环节的编码实现。

MapReduce的框架具有平滑无缝的可扩展性，利用硬件的扩展来处理增长的数据，并就近（数据）处理计算任务。在任务处理过程中，对于失效出错的节点进行无缝的接管。

![mapreduce_process](img/hadoop_mapreduce_process.png)

## 可由框架完成的工作有：

- 计算任务的划分和调度
- 数据的分布存储和划分
- 处理数据与计算任务的同步
- 结果数据的收集整理(sorting, combining, partitioning, grouping…)
- 系统通信、负载平衡、计算性能优化处理
- 处理系统节点出错检测和失效恢复

## 可供业务编码的工作有：

- InputFormat、InputSplit，输入数据与切分
- Mapper，map业务，可级联
- Partitioner，中间结果分区
- Combiner，中间结果合并，也是个Reducer
- Sort，中间结果排序，实现了RawComparator
- Grouping，中间结果分组，实现了RawComparator
- Reducer，reduce业务，可级联
- OutputFormat，输出数据

## Shuffle 数据混洗

Shuffle混洗过程主要是：数据分区，排序，局部聚合，缓存，拉取，再合并 排序。并在分发的过程中，对数据按 key 进行了分区和排序。为了把一组无规则的数据尽量转换成一组具有一定规则的数据，提高后续节点的处理效率。主要是解决性能问题，减少带宽的消耗，减少磁盘IO。

Reduce 的数据来源于 Map， Map 的输出即是 Reduce 的输入， Reduce 需要通过 Shuffle 来获取数据。从 Map 输出到 Reduce 输入的整个过程可以广义地称为 Shuffle。Shuffle 横跨 Map 端和 Reduce 端。

- Map阶段的shuffle过程：
  map中间结果写入“环形内存缓冲区”时进行**partition**分区。
  超过缓冲区阙值后**spill**溢写入到磁盘，写入前可以进行的是s**ort排序**与**combine**合并，这个过程每次一个spill_file文件。
  map结束时，对溢写的文件进行一次**merge**合并，输出最终结果文件。
- Reduce阶段的shuffle过程：
  reduce前从各节点上**copy**拉取map阶段的结果文件，放内存缓冲区中。
  超过缓冲区阙值后做**merge**合并，并**spill**溢写入到磁盘，这阶段也可以有**combine**合并操作。仍然每次一个文件。
  拉取map结果数据结束后做**merge**合并，并写入到磁盘文件做reduce的输入文件。

