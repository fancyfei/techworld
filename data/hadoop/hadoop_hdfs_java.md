# HDFS文件系统的使用

Java抽象类org.apache.hadoop.fs.FileSystem定义了hadoop的一个文件系统接口。Hadoop中关于文件操作类基本上全部是在"org.apache.hadoop.fs"包中，这些API能够支持的操作包含：打开文件，读写文件，删除文件等。

Hadoop类库中最终面向用户提供的接口类是FileSystem，该类是个抽象类，只能通过来类的get方法得到具体类。

## Jar包依赖

Jar包，需要保持与服务器安装版本保持同一版本。主要是hadoop-hdfs-client、hadoop-common等组件。或者引用hadoop-client组件，会自动加载上以上两个和yarn、mapreduce的Jar包。

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>3.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs-client</artifactId>
    <version>3.3.0</version>
</dependency>
```

## 初始化FileSystem

先配置Configuration，它是Hadoop的公共类，各类配置信息，如指定副本数、集群名等等。再就可以指定URI与user来初始化FileSystem，必须配置HDFS的网络路径URI，与fs.defaultFS中保持一致。

```java
String nameNodeUrl="hdfs://hadoop_server:9000";
Configuration configuration=new Configuration();
configuration.set("fs.defaultFS", nameNodeUrl);
configuration.set("dfs.client.use.datanode.hostname", "true");
URI uri = new URI(nameNodeUrl);
FileSystem fileSystem=FileSystem.get(uri,configuration,"root");
```

## 操作文件

初始化了FileSystem，就可以使用它的API完成文件的查看、新增、删除等等操作了。

比如写字符串到新文件：

```java
public void write(FileSystem fileSystem, String path, String content) {
    try {
        FSDataOutputStream out=fileSystem.create(new Path(path), true);
        ByteArrayInputStream in = new ByteArrayInputStream(content.getBytes());
        IOUtils.copyBytes(in, out, 1024,true);
    } catch (IOException e) {
    	e.printStackTrace();
    }
}
```

FileSystem常见的方法有：

- 构造方法，public static FileSystem.get(Configuration conf)和get(URI uri, Configuration conf)。
- 新建目录，public boolean mkdirs(Path path)，将会完整的创建path整个目录，包括父目录。
- 写入数据，public FSOutputStream create(Path file)，返回一个用于写入数据的输出流，create方法还可以指定是否强制覆盖已有的文件、文件备份数量、写入文件缓冲区大小、文件块大小以及文件权限等等参数。
- 删除，public boolean delete(Path f, Boolean recursive)。永久性删除指定的文件或目录，recursive指定是否删除非空目录。
- 读出文件内容，public FSDataInputStream open(Path file)，返回一个用于读出数据的输入流。
- 读出文件信息，public FileStatus getFileStatus(Path file)，可以读出文件或目录的信息，而不是内容。
- 列出文件，public FileStatus[] listStatus(Path f)，可以读出文件夹下所有的文件或子目录的信息。
- 重命名，public boolean rename(Path src, Path dst)。
- 是否存在，public boolean exists(Path f) ，检查文件或目录是否存在。
- 复制本地文件到网络文件，public void copyFromLocal(Path src, Path dst) 。
- 复制网络文件到本地文件，public void copyToLocal(Path src, Path dst) 。

