# 亚马逊的计算实例
亚马逊的计算实例，Amazon Elastic Compute Cloud (Amazon EC2)，讲人话就是咱们通常所说的虚机。
这也是公有云提供的最基础的功能之一，提供操作系统和一些可选的软件。
EC2是允许按需伸缩的。
按公布的消息来看，Xen是AWS上虚拟化技术的主体，最新的使用了Nitro技术（AWS的有专用硬件支持的更轻量虚拟平台）。

AWS上的计算类服务还有，无服务器式（Serverless）应用服务。

## 重要的功能
1. 操作系统，Linux的各种版本和Windows各种版本。需要版权的要收费。
2. 实例的CPU、内存、存储和网络容量等可选配置项。
3. 存储，临时的（实例存储卷）和持久性的（块存储卷，可多个位置的）。
4. 安全相关，密钥、防火墙等。
5. 网络相关，静态IP地址（弹性IP）、网络逻辑隔离（VPC）等。

## 镜像 Amazon Machine Images (AMI)
预配置的EC2镜像被称之为Amazon Machine Images (AMI)，一个AMI包含了打包好的操作系统，以及相应的应用程序和配置。
相当于一个做好的快照。多数使用亚马逊AMI market上提供的AMI即可，也可以制作自已符合自已业务的快照，保存在S3上。

## 存储的选择
EC2实例存储（Instance store volumes）是一种短暂性的存储，一旦停止或者终止EC2实例，这个存储内的数据将永久消失。

EBS存储（Amazon EBS volumes）是一种持续性的存储，不管EC2实例是什么状态，你都可以保留EBS存储内的数据（可选的，也可以选择终止实例时删除存储）。
这种类型的存储对于进行数据盘的迁移非常方便，使用场景也比较多，多数常用的AMI的根卷仅支持EBS。

更多的文件共享可以选择EFS储存，或者S3存储。

## 网络设置
私有 IPv4 地址，默认情况下，Amazon EC2 和 Amazon VPC 使用 IPv4 寻址协议。
启动实例时，会为实例分配一个私有 IPv4 地址（或指定）。另外，还为每个实例指定了一个可解析这个 IPv4 地址的内部 DNS 主机名。

公有 IPv4 地址，通过 Internet 可访问的 IPv4 地址。同样也会有一个外部 DNS 主机名，AWS会解析这个主机名到此IPv4 地址。
以及其 VPC 内的实例的私有 IPv4 地址。公有 IP 地址是通过网络地址转换 (NAT) 映射到对应私有 IP 地址。
一旦在取消公有 IP 地址与实例的关联后，该地址即会释放。

弹性IP（Elastic IP address）可以方便为EC2实例分配一个固定的公网IP地址，重启依旧有效。

虚拟私有云（Virtual Private Cloud, VPC）是AWS的网络组件，主要是逻辑上隔离资源，也可以使用VPC与自已的数据中心进行连接。

Amazon DNS 服务器，可将 Amazon 提供的 IPv4 DNS 主机名解析为 IPv4 地址。

## 接入
AWS EC2管理平台 – AWS提供的web用户界面来访问EC2实例（通过java等插件）。

AWS CLI工具 – 可以通过AWS CLI工具来访问AWS的多个组件。

SSH工具可以连接上去。

还有AWS API 、AWS SDK。

## 计费类型
On-Demand Instances （按需实例），提供以Gb/小时为计费单位，无须预付费，无需关注时长。
Reserved Instances （预留实例），在按需实例基础上，加入预留机制（承诺使用时长）。成本低50%。
Spot Instances （竞价实例），自定价格，租用EC2服务的闲散资源。需要关注自定价格低于即时价格时，系统自动终止服务。

还另外几种计费类型，
Scheduled Reserved Instances （计划的预留实例）
Dedicated Instances （专用的实例）
Dedicated Hosts（专用的主机）
