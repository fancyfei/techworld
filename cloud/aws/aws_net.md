#  AWS服务网络架构
AWS向客户所提供网络服务（Network As A Service）以及各种不同的服务须要的网络架构。
AWS所抽象过的网络组件，隐藏了很多现实中网络的细节。

较常见的架构，由VPC构建出AWS里面虚拟局域网，在同一个VPC下的划子网(subnet)。
子网通过互联网关（Internet Gateways）连通外网，如果子网中多EC2使用同一个互联网关，则可以通过NAT网关中转，访问规则记录在路由表（Routing Tables）。
子网之间可默认可访问，但是可以对ACL和Security Group进行访问控制。

## 常见的网络及访问控制
- 互联网访问，通过IGW互联网关（Internet Gateways）。
- VPC内网，默认可访问，权限由ACL和SG（Security Group）控制。
- REGION下不同VPC之间，借助VPC Peering来实现互联。
- 不同REGION的VPC之间，则需要公网的VPN来实现IPSec（安全协议）连接，需要网段不重。
- VPC和客户内网之间，同样要公网的VPN，另外为保证质量可用AWS的VPN，如需要无障碍打通需要专线。

## VPC提供的网络组件
- VPC

  VPC （Virtual Private Cloud ），虚拟私有云，是基础设施所运行在的一个私有网络空间。VPC有一段自已的地址空间（CIDR 范围），例如 10.0.0.0/16 。这将决定 VPC 内有多少可分配 IP 。相互之间是隔离的网络环境。

  每个用户都有一个默认的VPC，每个VPC下都有一个默认子网，VPC创建的时候会有一个默认主路由表。在VPC内部无需使用 Internet 网关、NAT 或防火墙代理。

- Subnet（子网）

  子网是VPC的一部分，具有自己的CIDR范围以及流量如何流动的规则（路由表），每个子网都有其自己的规则。能被外网访问的是“公有”子网，不能被外网访问的是“私有”子网。Subnet在默认下会关联到VPC的主路由表。将默认路由指向一个NAT网关，“私有”子网就可以通过NAT网关访问互联网。

- Route Table（路由表）

  路由表规定了一系列子网 IP 数据包如何传输到其他不同 IP 地址的规则。创建VPC时就会自动生成一个主路由表，默认情况下子网将与主路由表建立隐式关联。可以为 VPC 创建多个额外的自定义路由表，显式控制每个子网的出站数据流的路由方式。

- Internet Gateway（互联网关）

  互联网关是VPC 中的实例访问Internet的必经之地。在 VPC 路由表中路由目标设定为互联网关则可访问外网；互联网关将Internet流量进行NAT到有公有 IP 地址的实例。

  将 Internet 网关附加到 VPC，子网与公网的路由表（包含指向 Internet 网关的路由）相关联，该子网就是所谓的公有子网，配置安全组与访问控制允许流入和流出，则子网中的实例（有全局唯一 IP 地址）就可以被Internet 访问到了。

- NAT (网络地址转换) 

  使用 NAT 设备允许私有子网中的实例连接到 Internet 或其他 AWS 服务，但阻止 Internet 发起与实例的连接。

- SG & Network ACL

  



