# 亚马逊的云存储方案
亚马逊AWS给用户提供了不同种类的云存储方案，不同的存储系统有不一样的特性和优劣。
大家比较常用的是实例存储、S3、EBS、EFS。

在EC2的根设备存储由 Amazon EBS 支持或由实例存储支持。这得看选择的Amazon 系统映像 (AMI) 支持的是哪种。
启动时将自动挂载根卷。启动后可以附加额外的 EBS 卷，但是不可以附加实例存储卷。

## Amazon 实例存储（instance store）
Amazon EC2 实例存储，实例存储为实例提供临时性块级存储。此存储位于已物理附加到主机的磁盘上。
它的生命周期是启动实例时，数据一直有效；当故障时、停止、终止时，数据就消失。
实例启动后不可以附加和变更。可选HDD、SSD与NVMe SSD（非易失性存储）。
实例存储中，交换空间可以单独设置存储卷。

## AWS Elastic Block Store (EBS)
AWS EBS是可以用来做数据库或托管应用程序的持续性文件系统（块存储），EBS具有很高的IO读写速度并且即插即用。
仅用于 EC2 实例。提供以下卷类型：通用型 SSD (gp2)、预配置 IOPS SSD (io1)、吞吐优化 HDD (st1) 和 Cold HDD (sc1)。

其中的预配置 IOPS SSD (io1)，是最高性能 SSD 卷，高并发的数据库可选择此类型。
在HDD类型中，最低成本的Cold HDD (sc1)可以保存一些不经常访问的数据，成本只有io1的1/5。
一般正常业务应用，不需要高IO的，普通的吞吐优化 HDD (st1)就够了。但是EC2的根卷多数AMI只支持SSD (gp2)。

EBS是支持弹性的，通过使用 Amazon EBS 弹性卷，您可以增加卷大小，更改卷类型或调整 EBS 卷的性能。

EBS支持快照，将 Amazon EBS 卷上的数据备份到 Amazon S3。可以定期增量快照，第一次将全量快照。

## Simple Storage Service (S3) 
AWS S3，对象存储服务，对于静态页面的托管、多媒体分发、版本管理、大数据分析、数据存档来说都非常有用。
Amazon S3 提供了可靠、快速和廉价的数据存储基础设施。可以使用 API 从互联网访问 Amazon S3 中的数据。
S3可以和AWS CloudFront结合使用而达到更快的上传和下载速度。

Amazon EC2 还使用 Amazon S3 来存储数据卷的快照。

## Elastic File System (EFS)
AWS EFS提供了可以在多个EC2实例之间共享的网络文件系统，功能类似于NAS设备，可扩展文件存储。
可以通过网络文件系统 (NFSv4) 协议在 VPC 中挂载 Amazon EFS 文件系统。

可以用EFS来处理大数据分析、多媒体处理和内容管理。
特点是高可用的文件系统，但是成本比较高，大约是EBS的SSD存储类型的3倍。

Amazon EFS 提供两个存储类：标准存储类和不频繁访问存储类 (EFS IA)。IA的存储成本只有约1/3。
在启动生命周期管理后，将在一段设定时间内未访问的文件迁移到不常访问 (IA) 存储类别。

## AWS 存储的关系
下图显示了这些存储选项和实例之间的关系。
![AWS的储存](img/aws_architecture_storage.png)