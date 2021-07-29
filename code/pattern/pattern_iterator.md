# 迭代器模式

迭代器（Iterator）模式，是一种对象行为型模式，提供一种方法顺序访问一个聚合对象中的各个元素，而又不需要暴露该对象的内部表示。

迭代器模式主要场景是处理集合的顺序访问，统一抽象为迭代器。

集合具体的数据结构有简单列表存储，也有栈、 树、 图和其他复杂的数据结构。不同的数据结构，遍历元素的方式也不同，将集合的遍历行为抽取为单独的迭代器对象，可以让客户端不用关心具体的实现细节。

在Java的集合类中被广泛使用。

## 迭代器模式的实现

迭代器模式实现的关键就是迭代器的抽象，主要角色如下：

- 抽象聚合（Collection）角色：聚合对象的接口。
- 具体聚合（Concrete Collection）角色：实现抽象聚合类。
- 抽象迭代器（Iterator）角色：定义访问和遍历聚合元素的接口，通常包含 hasNext()、first()、next() 等方法。
- 具体迭代器（Concrete lterator）角色：实现抽象迭代器接口中所定义的方法，完成对聚合对象的遍历，记录遍历的当前位置。

类图如下：

![pattern_iterator](pattern_iterator.png)

代码比较简单，在JDK中，ArrayList与List，ArrayList的内部类Itr与Iterator就组成简单的迭代器模式。

```java
//迭代器接口
public interface Iterator<E> {
    boolean hasNext();
    E next();
}
//集合接口
public interface List<E> extends Iterable<E>{
    Iterator<E> iterator();
}
//集合实现
public class ArrayList<E> implements List<E>{
    private int size;
    public Iterator<E> iterator() {
        return new Itr();
    }
    //内部类实现迭代器
    private class Itr implements Iterator<E> {
        int cursor;
        int lastRet = -1;
        public boolean hasNext() {
            return cursor != size;
        }
        public E next() {
            int i = cursor;
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
    }
}
```

