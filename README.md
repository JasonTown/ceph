# Ceph：从入门到反复入门

## 1. 概述

### 1.1 定义

Ceph是一种为优秀的性能、可靠性和可扩展性而设计的统一的、开源的**分布式的存储系统**。

详细内容可以参考[官方文档](http://docs.ceph.org.cn/)

### 1.2 特点

- Ceph提供给了**对象、块、文件存储**功能，适合海量数据的管理。
- 有强大的伸缩性：供用户访问PB乃至EB级的数据
- Ceph存储集群织起了大量节点，它们之间靠相互通讯来复制数据、并动态地重分布数据。

### 1.3 核心组件

Ceph的核心组件包括**Ceph OSD**、**Ceph Monitor**、**Ceph MDS**三大组件。

- **Ceph OSD**

Object Storage Device，主要功能是存储数据、复制数据、平衡数据、恢复数据等，与其他OSD进行心跳检查，上报变化情况给Ceph Monitor。一般一个OSD对应一个硬盘（或分区），并对其进行管理。

- **Ceph Monitor**

负责监视Ceph集群，维护Ceph集群的健康状态、Cluster Map。

> Cluster Map是RADOS的关键数据结构，管理集群种的所有成员、关系、属性等信息，以及数据分发。
>
> 当用户需要存储数据到Ceph集群时，OSD先通过Monitor获取最新Map图，再根据Map图和Object id等，计算出数据最终存储位置。

- **Ceph MDS**

Ceph MetaData Server，主要保存的文件系统服务的元数据，但对象存储和块存储设备是不需要使用该服务的。

> 元数据：
>
> 又称为中介数据、中继数据。主要描述数据属性的信息，用来支持如指示存储位置、历史数据、资源查找、文件记录等功能。简言之，就是**用于描述一个文件特征的系统数据**，比如访问权限、文件拥有者以及文件数据库的分布信息（inode)等等。在集群文件系统中，分布信息包括文件在磁盘上的位置以 及磁盘在集群中的位置。用户需要操作一个文件就必须首先得到它的元数据，才能定位到文件的位置并且得到文件的内容或相关属性。

- **Managers**

Ceph Manager守护进程（ceph-mgr）负责跟踪运行时指标和Ceph集群的当前状态，包括存储利用率，当前性能指标和系统负载。

> Ceph Manager守护进程还基于python的插件来管理和公开Ceph集群信息，包括基于Web的Ceph Manager Dashboard和 REST API。高可用性通常至少需要两个管理器。



### 1.4 架构

![架构图](https://s2.ax1x.com/2020/03/05/37683j.png)

Ceph可分为四个层级

- **（1）基础存储系统RADOS**

RADOS (Reliable, Autonomic, Distributed Object Store) 意为可靠、自动、分布式的对象存储，这一层是Ceph的基础，即用来存储用户数据一层。

- **（2）基础库librados**

对RADOS抽象、封装，向上层提供API，便于直接基于RADOS进行开发。

> RADOS采用C++开发，原生librados API包括C和C++两类。librados和基于其开发的应用在物理上位于同一台机器，也因此被称为本地API。应用通过本地调用librados API，再由librados API通过socket与RADOS集群中的节点通信完成操作。

- **（3）高层应用接口**

  - **RADOS GW (RADOS Gateway)** ：是一个提供与Amazon S3和Swift兼容的RESTful API的gateway，以供相应的对象存储应用开发使用。RADOS GW提供的API抽象层次更高，但功能则不如librados强大。因此，开发者应针对自己的需求选择使用。

  - **RBD (Reliable Block Device)**：提供了一个标准的块设备接口，常用于在虚拟化的场景下为虚拟机创建volume。目前，Red Hat已经将RBD驱动集成在KVM/QEMU中，以提高虚拟机访问性能。

- **（4）Ceph FS **

Ceph FS(Ceph File System) ，即Ceph集群的文件管理系统。通过Linux内核客户端和FUSE来提供一个兼容POSIX的文件系统。



**RADOS的存储逻辑架构**

![RADOS系统结构图](https://s2.ax1x.com/2020/03/05/372cKs.png)

RADOS集群主要有两种节点：

1. OSD（负责数据存储和维护）
2. Monitor（系统状态检测和维护）

OSD和Monitor之间相互传输节点状态信息，形成Cluster Map。Cluster Map这个数据结构和RADOS提供的特定算法相结合，实现了Ceph“无需查表，算算就好”的核心机制和若干优秀特性。

> [如何“算”出存储位置：CRUSH](https://zhuanlan.zhihu.com/p/63725901)

**客户端程序通过Cluster Map获取节点状态信息，然后在本地计算出OSD位置，之后直接与OSD通信，完成操作。**

由此可见，只要保证Cluster Map不频繁更新，则客户端完全不需要查表，也不依赖于任何元数据服务器，就可以访问OSD。RADOS运行过程中，Cluster Map的更新取决于系统状态的变化，只有两种：

1. OSD出现故障。
2. RADOS规模扩大。正常情况下，这两种事件发生的频率显然远远低于客户端对数据进行访问的频率。



---

## 2. Ceph工作原理及流程

### 2.1 RADOS对象寻址

Ceph存储集群，从Ceph客户端接收数据并存储为对象。对象是文件系统中的一个文件，存储在OSD上，由Ceph OSD守护进程处理OSD上的读/写操作。通常来说，一块磁盘和该磁盘对应的守护进程称为一个OSD。

传统架构中，客户端与一个中心化组件通信（网关、API、中间件等），可能导致单点故障并限制了性能与伸缩性。

Ceph消除了集中网关，允许客户端直接和Ceph OSD守护进程通讯。同时Ceph会自动在其他Ceph节点创建对象副本来确保数据安全和高可用性。此外，为了保证高可用性，Ceph Monitor也实现了集群化。该功能基于CRUSH算法。

![Ceph寻址流程图](https://s2.ax1x.com/2020/03/08/3vIpTK.png)



- File

用户存储/访问的文件。用户存储数据到Ceph集群时，存储数据会被分割成多个object。

- object

Ceph存储的最小存储单元。每个object都有一个object id，每个object的大小是可以设置的，默认是4MB。

- PG

Placement Group，对object的存储进行组织和位置映射。PG是一个逻辑概念，类似于数据库中的索引表，它的数量是固定的，不会随着OSD增加或减少而改变。每个object最后都会通过CRUSH计算映射到某个PG中，一个PG可以包含多个object。而PG和OSD之间则是“多对多”的映射关系。进行数据迁移时，Ceph会以PG为单位进行迁移，不会直接操作对象。PG存在的意义是提高ceph存储系统的性能和扩展性。

> PG是一种间址，PG的数量有限，记录PG跟OSD间的映射关系可行，而记录object到OSD之间的映射因为数量巨大而实际不可行或效率太低。从用途来说，搞个映射本身不是目的，**让故障或者负载均衡变得可操作是目的。**

- OSD

Object Storage Device，PG通过CRUSH算法映射到OSD中存储。如果是二副本的，则每个PG都会映射到二个osd，比如[osd.1,osd.2]，那么osd.1是存放该PG的主副本，osd.2是存放该PG的从副本，保证了数据的冗余。



**流程**

![结构](https://s2.ax1x.com/2020/03/10/8FDSJg.jpg)

- 将file切片为多个object，每个object有一个oid。

> 切割规则本质上就是按照object的大小来切file，优点：
>
> 1. object大小一致便于RADOS高效管理
> 2. file的串行处理变成object的并行处理

> oid = ino (identity number) + ono (object number)，ino是file的id，ono是object的id，前者用于文件定位，后者用于文件切片位置的定位。

- 一个或多个object映射到PG中，映射规则：`oid = hash(oid) % m`

> ceph利用静态hash函数将oid集合映射为近似均匀分布的伪随机值集合，并且集合中每个oid都与PG的数量去模得到PG序号 (PGid)。PG的数量多寡直接决定了数据分布的均匀性，所以合理设置的PG数量可以很好的提升CEPH集群的性能并使数据均匀分布。
>
> 1. hash完成均匀分布
> 2. 取模随机选取
>
> 因为只有大量object和PG，这种伪随机关系的近似均匀才能成立，故要注意两点：
>
> 1. object的size要合理配置，使得file能被切分成足够多的object
> 2. PG数量应该为OSD总数的数百倍，才能保证有足够数量的PG提供映射。

- 一个PG映射到n个OSD中（称为n副本，常用3副本），PGid通过CRUSH计算，得到n个OSD，其上运行的OSD deamon负责执行映射到本地的object在本地文件系统中的存储、访问、元数据维护等操作。



### 2.2 数据的操作流程

模拟file的写入流程，为便于理解先假设如下前提：

1. file较小，无需切分，故被映射到一个object上
2. 三副本情况

![file写入流程](https://s2.ax1x.com/2020/03/09/8pnLqS.png)

**流程描述**

- 本地RADOS对象寻址，得到OSDs的id

client向ceph集群写入file时，先在本地进行RADOS对象寻址，将file转为object，并得到一组的3个OSDid，按照序号大小，分别为Primary OSD, Secondary OSD, Tertiary OSD。

- 发起写入操作

client直接与Primary OSD通信，发起写入操作，Primary OSD接收到之后，并不立即执行写入操作，而先向Secondary OSD, Tertiary OSD发起写入操作，等待确认信息。

- 回复确认信息

Secondary OSD和Tertiary OSD完成写入操作后，回复Primary OSD确认信息，Primary OSD才执行写入操作，并回复client确认信息，至此完成写入操作。

> 等待确认信息的过程会有较大的延时，因此，ceph可以分两次向client回复确认信息。
>
> 1. 第一次，数据写入OSD内存缓冲区时，回复client。此时client可继续执行操作。
> 2. 第二次，数据从缓存写入硬盘后，再回复client。此时client可根据需要删除本地数据，从而完成写入操作。



从上述流程可以看到，client寻址直接在本地完成，无需中心化组件。故client与OSD之间可以多对多并行通信，满足了分布式的要求。并且，OSD在各个PG中可能担任的角色不同，从而尽最大可能均分了工作负担，避免单个OSD的性能问题造成系统的性能瓶颈。

读取数据同理，client直接和Primary OSD通信，最终由Primary OSD返回读数据。

> Q&A：
>
> 1. 读数据是否可以从任意从OSD读取而非仅仅从主OSD读取？
> 2. 或者是否可以从n个OSD并行读取呢？从而提高带宽利用率。



### 2.3 集群维护

若干Monitor共同负责了Ceph集群所有OSD状态的发现与记录，并形成Cluster Map，并分发到各个OSD和client中。OSD利用Cluster Map进行数据的维护，client利用Cluster Map进行寻址操作。

Monitor之间为主从备份关系，Monitor并不主动轮询OSD（这样做时间开销太大），而是OSD主动上报状态信息。Monitor收到信息后更新Cluster Map。触发条件有：

1. 新OSD加入集群
2. 某个OSD发现自身
3. 其他OSD出现异常



**Cluster Map的组成**

- Monitor Map

包含集群的 `fsid` 、位置、名字、地址和端口，也包括当前版本、创建时间、最近修改时间。要查看监视器图，用 `ceph mon dump` 命令。

- OSD Map

包含集群 `fsid` 、创建时间、最近修改时间、存储池列表、副本数量、归置组数量、 OSD 列表及其状态（如 `up` 、 `in` ）。要查看OSD运行图，用 `ceph osd dump` 命令。

OSD状态的描述分为两个维度：up或者down（表明OSD是否正常工作），in或者out（表明OSD是否承载PG）。因此，对于任意一个OSD，共有四种可能的状态：

—— Up且in：说明该OSD正常运行，且已经承载至少一个PG的数据。这是一个OSD的标准工作状态；

—— Up且out：说明该OSD正常运行，但并未承载任何PG，其中也没有数据。一个新的OSD刚刚被加入Ceph集群后，便会处于这一状态。而一个出现故障的OSD被修复后，重新加入Ceph集群时，也是处于这一状态；

—— Down且in：说明该OSD发生异常，但仍然承载着至少一个PG，其中仍然存储着数据。这种状态下的OSD刚刚被发现存在异常，可能仍能恢复正常，也可能会彻底无法工作；

—— Down且out：说明该OSD已经彻底发生故障，且已经不再承载任何PG。

- PG Map

包含归置组版本、其时间戳、最新的 OSD 运行图版本、占满率、以及各归置组详情，像归置组 ID 、 up set 、 acting set 、 PG 状态（如 `active+clean` ），和各存储池的数据使用情况统计。

- CRUSH Map

包含存储设备列表、故障域树状结构（如设备、主机、机架、行、房间、等等）、和存储数据时如何利用此树状结构的规则。要查看 CRUSH 规则，执行 `ceph osd getcrushmap -o {filename}` 命令；然后用 `crushtool -d {comp-crushmap-filename} -o {decomp-crushmap-filename}` 反编译；然后就可以用 `cat` 或编辑器查看了。

- MDS Map

包含当前 MDS 图的版本、创建时间、最近修改时间，还包含了存储元数据的存储池、元数据服务器列表、还有哪些元数据服务器是 `up` 且 `in` 的。要查看 MDS 图，执行 `ceph mds dump` 。



### 2.4 增加OSD流程

- 新OSD上线后，如果PG处于正常状态，则新OSD首先根据配置信息与monitor通信。Monitor将其加入cluster map，并设置为up且out状态，再将最新版本的cluster map发给这个新OSD。收到monitor发来的cluster map之后，这个新OSD计算出自己所承载的PG（为简化讨论，此处我们假定这个新的OSD开始只承载一个PG），以及和自己承载同一个PG的其他OSD。然后，新OSD将与这些OSD取得联系。如图：

![PG处于正常状态，OSD未启用](https://s2.ax1x.com/2020/03/09/8pYyu9.png)

- 新OSD上线后，如果PG处于降级状态（即承载该PG的OSD个数少于正常值，如正常应该是3个，此时只有2个或1个。这种情况通常是OSD故障所致），则其他OSD将把这个PG内的所有对象和元数据复制给新OSD。数据复制完成后，新OSD被置为up且in状态。而cluster map内容也将据此更新。这事实上是一个自动化的failure recovery过程。当然，即便没有新的OSD加入，降级的PG也将计算出其他OSD实现failure recovery，如图：

![PG处于降级状态时](https://s2.ax1x.com/2020/03/09/8ptQV1.png)

- 如果该PG目前一切正常，则这个新OSD将替换掉现有OSD中的一个（PG内将重新选出Primary OSD），并承担其数据。在数据复制完成后，新OSD被置为up且in状态，而被替换的OSD将退出该PG（但状态通常仍然为up且in，因为还要承载其他PG）。而cluster map内容也将据此更新。这事实上是一个自动化的数据re-balancing过程，如图：

![PG处于正常状态，新旧OSD更换](https://s2.ax1x.com/2020/03/09/8ptaqA.png)

---

## 3. Ceph (Luminous)集群创建

一个Ceph集群至少需要3个OSD才能实现冗余和高可用性，并且至少需要一个mon和一个mds。其中mon和mds可以部署在其中一个osd节点上，详细可参考[官方安装文档](http://docs.ceph.org.cn/start/)。

### 3.1 服务器初始化



**不要用CentOS 8！！！**

**不要用CentOS 8！！！**

**不要用CentOS 8！！！**

**不要问我怎么知道的，全是泪……**



官方文档给出了建议的[操作系统](http://docs.ceph.org.cn/start/os-recommendations/#id2)，不看官方文档的后果就是…吐血

本来想用虚拟机的，后来想到我在Vultr上还有100$赠送券没用，索性直接用VPS来实现了。

结果Vultr给我埋了个坑！厂商初始化的系统，只有一个vda磁盘，且所有的容量都挂载根分区了。而Ceph需要一个格式为xfs的单独分区，搜了下linux缩小根分区要进rescue模式，VPS又不好弄…

解铃还须系铃人，搜了半天，最终找到解决办法了。就是使用VPS厂商提供的**Block Storage**功能来扩展出一个分区。不过最蛋疼的是这个功能支持**纽约**机房的VPS，之前搞了半天西雅图的机房，又得重来...裂开了

扩展好分区之后，硬盘情况如下：

![硬盘分区情况](https://s2.ax1x.com/2020/03/10/8i3wwR.png)

各个节点配置如下：

| 主机名      | 系统         | IP              | 数据磁盘 | 备注          |
| ----------- | ------------ | --------------- | -------- | ------------- |
| ceph-client | CentOS 7 X64 | 140.82.15.88    | 25G      | client        |
| ceph-node1  | CentOS 7 X64 | 104.207.133.105 | 25G      | mgr.mon.osd.0 |
| ceph-node2  | CentOS 7 X64 | 45.63.5.140     | 25G      | osd.1         |
| ceph-node3  | CentOS 7 X64 | 149.28.40.16    | 25G      | osd.2         |

> mgr：管理节点
>
> mon：监控节点
>
> osd：存储节点
>
> Ceph集群要求必须是奇数个mon监控节点，一般建议至少是3个
>
> Ceph的文件系统作为一个目录挂载到客户端cephclient的``/cephfs``目录下，可以像操作普通目录一样对此目录进行操作。



### 3.2 环境准备

ok，话不多说，先把环境配好。

#### (1) 修改hostname并绑定主机名映射

这一步是为了统一名称，方便后面命令操作使用的。

**在4个主机上修改hostname**

```shell
[root@ceph1 ~]# hostnamectl set-hostname ceph-client
[root@ceph2 ~]# hostnamectl set-hostname ceph-node1
[root@ceph3 ~]# hostnamectl set-hostname ceph-node2
[root@ceph4 ~]# hostnamectl set-hostname ceph-node3
```

> 如果使用ssh登陆的远程VPS，设置之后可能没有刷新hostname，exit退出重新登陆即可



**在3个节点机上添加hosts**

```shell
[root@ceph-node1 ~]# vi /etc/hosts

104.207.133.105 ceph-node1
45.63.5.140 ceph-node2
149.28.40.16 ceph-node3
```

 连通性测试

```shell
[root@ceph-client ~]# ping -c 2 ceph-node1
[root@ceph-client ~]# ping -c 2 ceph-node2
[root@ceph-client ~]# ping -c 2 ceph-node3
```

![连通性测试成功](https://s2.ax1x.com/2020/03/10/8CgorR.png)

另外2个节点机同理执行如上操作



#### (2) 创建ceph用户

我们一般不会再root下工作，所以创建一个`ceph`用户来使用。

**在3个节点机上创建用户ceph，密码统一为ceph**

```shell
[root@ceph-node1 ~]# adduser ceph && echo "ceph:ceph"|chpasswd
```

为每个ceph节点用户增加root权限

```shell
[root@ceph-node1 ~]# echo "ceph ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph && chmod 0440 /etc/sudoers.d/ceph
```

测试root权限，`exit`退回切换前用户

```shell
[root@ceph-node1 ~]# su - ceph 
[ceph@ceph-node1 ~]# sudo su -
```



#### (3) 安装NTP

建议3个节点机安装NTP，以免因时钟漂移导致故障（mon节点必须）

```shell
[root@ceph-node1 ~]# yum install ntp ntpdate ntp-doc -y 
[root@ceph-node1 ~]# systemctl restart ntpd
[root@ceph-node1 ~]# systemctl status ntpd
```



#### (4) 关闭防火墙和Selinux

保证各个节点之间的联通，在开发阶段完全可以关闭防火墙，之后再重新打开配置。

SELinux是一种基于 域-类型 模型（domain-type）的强制访问控制（MAC）安全系统，说人话就是Linux的一个安全子系统，它是默认开启的 ，会对我们Ceph安装过程造成一定的影响，所以先关掉它。

**关闭防火墙**

```shell
[root@ceph-node1 ~]# systemctl stop firewalld
[root@ceph-node1 ~]# systemctl disable firewalld
```



**关闭selinux**

```shell
[root@ceph-node1 ~]# sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config && setenforce 0
```



#### (5) epel源配置

yum默认的源是国外的，因为众所周知的原因，它的速度可能会比较慢。所以我们换成国内的源。

**把软件包源加入软件仓库**

用文本编辑器创建一个 YUM 库文件：

```shell
[root@ceph-node1 ~]# vi /etc/yum.repos.d/ceph.repo

[ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/$basearch
enabled=1
gpgcheck=1
priority=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/noarch
enabled=1
gpgcheck=1
priority=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/SRPMS
enabled=0
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
priority=1
```

缓存服务器的包信息

```shell
[root@ceph-node1 ~]# yum makecache
```

更新yum

```shell
[root@ceph-node1 ~]# yum update -y
```



#### (5) mgr节点安装ceph-deploy

之前的操作，都需要在所有非客户端节点进行，这个只需要在作为mgr节点的`ceph-node1`上操作即可。

**安装ceph-deploy管理工具**

```shell
[root@ceph-node1 ~]# yum install -y ceph-deploy
```

![ceph-deploy部署工具安装成功](https://s2.ax1x.com/2020/03/10/8PmlKx.png)



#### (6) 配置SSH服务器

正因为 `ceph-deploy` 不支持输入密码，你必须在管理节点上生成 SSH 密钥并把其公钥分发到各 Ceph 节点，完成节点间的免密码登陆。

**管理节点添加到各个节点机之间配置ssh服务器**

根据官方文档建议，不要使用`root`用户或`sudo`，切换回`ceph`用户。

```shell
[root@ceph-node1 ~]$ su - ceph
```

新建密钥（一路回车即可）

```shell
[ceph@ceph-node1 ~]$ ssh-keygen
```

公钥分发，期间提示输入其他节点`ceph`用户的密码，我们同样也是配置的`ceph`

```shell
[ceph@ceph-node1 ~]$ ssh-copy-id  ceph@ceph-node2
[ceph@ceph-node1 ~]$ ssh-copy-id  ceph@ceph-node3
```



**配置.ssh/config文件**

修改 ceph-deploy 管理节点`ceph-node1`上的 `~/.ssh/config `文件，这样 ceph-deploy 就能用你所建的用户名登录 Ceph 节点了，而无需每次执行 ceph-deploy 都要指定 –username {username} 。这样做同时也简化了 ssh 的用法。

```shell
[ceph@ceph-node1 ~]$ vi ~/.ssh/config

Host node1
   Hostname ceph-node1
   User ceph
Host node2
   Hostname ceph-node2
   User ceph
Host node3
   Hostname ceph-node3
   User ceph
```

> Host后为ssh连接的名称
>
> Hostname接host文件中的名字，也是ceph-deploy所用的名字

重启ssh服务（需要输入root权限的密码）

```shell
[ceph@ceph-node1 ~]$ sudo chmod 600 .ssh/config && systemctl restart sshd
```

测试ssh连接

```shell
[ceph@ceph-node1 ~]$ ssh node2
[ceph@ceph-node2 ~]$ exit
```



### 3.3 创建集群

#### (1) 创建ceph工作目录并配置ceph.conf

由于`ceph-deploy`会在当前目录产生文件，故新建一个目录。

**在mgr节点上创建ceph工作目录**

```shell
[ceph@ceph-node1 ~]$ mkdir my-cluster && cd my-cluster
```

创建ceph集群，`new`后接mon节点的主机名。

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy new ceph-node1
```

![安装成功](https://s2.ax1x.com/2020/03/10/8iyORe.png)

> 如果遇到python的import error：
>
> 这个问题通常是由于升级到python2.7后执行pip产生的，解决方案是重新在python2.7环境中安装pip，或者安装setuptools即可：

```shell
[ceph@ceph-node1 ~]$ sudo yum install -y python-setuptools
```



`ceph-deploy`的`new`子命令能够部署一个默认名称为`ceph`的新集群，并且它能生成集群配置文件和密钥文件。列出当前工作目录，你会看到`ceph.conf`和`ceph.mon.keyring`文件。

![产生的文件](https://s2.ax1x.com/2020/03/10/8icPmR.png)

> 如果需要(新安装的系统通常不需要)，部署之前确保ceph每个节点没有ceph数据包（先清空之前所有的ceph数据，如果是新装不用执行此步骤，如果是重新部署的话也执行下面的命令）

```shell
[ceph@ceph-node1]$ ceph-deploy purge ceph-deploy ceph-u0-m0 ceph-u0-l0 ceph-u0-r0
[ceph@ceph-node1]$ ceph-deploy purgedata ceph-deploy ceph-u0-m0 ceph-u0-l0 ceph-u0-r0
[ceph@ceph-node1]$ ceph-deploy forgetkeys
```



**配置监控节点ceph.conf**

```shell
[ceph@ceph-node1 cluster ~]$ vi ceph.conf
# 修改mon_host
mon_host = 104.207.133.105		//监控节点ip
```

[ceph.conf的配置文档](http://docs.ceph.org.cn/rados/configuration/ceph-conf/)



#### (2) 节点安装ceph

在管理节点，利用ceph-deploy工具为各个节点安装ceph

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy install ceph-node1 ceph-node2 ceph-node3
```

在各个节点检查安装情况

```shell
[ceph@ceph-node1 ~]$ ceph --version
[ceph@ceph-node2 ~]$ ceph --version
[ceph@ceph-node3 ~]$ ceph --version
```

![安装情况检查](https://s2.ax1x.com/2020/03/10/8iWt5q.png)



#### (3) mon节点初始化

初始化的时候，会生成mon节点监测集群所使用的秘钥

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy mon create-initial
```

> 本文在管理节点安装的mon，如果需要在其他节点安装mon，可使用如下命令：

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy mon node1 node2 node3 node4
```



**将管理节点的配置分发到其它节点**

收集所有密钥，OSD添加好后，用 `ceph-deploy` 把配置文件和 管理节点密钥拷贝到管理节点和 Ceph 节点，这样每次执行 Ceph 命令行时就无需指定 mon节点 地址和 `ceph.client.admin.keyring` 了

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy admin ceph-node1 ceph-node2 ceph-node3
```



将mon节点生成的密钥环分发到各个节点

```shell
[ceph@ceph-node1 cluster ~]$ sudo scp *.keyring ceph-node1:/etc/ceph/
[ceph@ceph-node1 cluster ~]$ sudo scp *.keyring ceph-node2:/etc/ceph/
[ceph@ceph-node1 cluster ~]$ sudo scp *.keyring ceph-node3:/etc/ceph/
```

允许`ceph`用户访问`etc/ceph`目录下的文件

```shell
sudo chown -R ceph:ceph /etc/ceph
```

输入`ceph -s`查看当前集群的状态信息

![ceph集群状态](https://s2.ax1x.com/2020/03/11/8AFMNQ.png)



#### (4) 配置mgr

mgr (manager)，用于管理集群

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy mgr create ceph-node1
```

开启仪表盘

```shell
[ceph@ceph-node1 cluster ~]$ ceph mgr module enable dashboard
```

通过`http://104.207.133.105:7000/`查看集群信息

![管理界面](https://s2.ax1x.com/2020/03/11/8AVFVf.png)



#### (5) 添加OSD到集群

**检查OSD节点上所有可用的磁盘**

```shell
[ceph@ceph-1 cluster]$ ceph-deploy disk list ceph-node1 ceph-node2 ceph-node3
```



**使用zap选项删除所有osd节点上的分区**

```shell
[ceph@ceph-1 cluster]$ ceph-deploy disk zap ceph-node1 /dev/vdb
[ceph@ceph-1 cluster]$ ceph-deploy disk zap ceph-node2 /dev/vdb
[ceph@ceph-1 cluster]$ ceph-deploy disk zap ceph-node3 /dev/vdb
```

> Ceph12版本以后prepare和activate命令已经没有了，用create替代

准备OSD（使用prepare命令）

```shell
[ceph@ceph-1 cluster]$ ceph-deploy osd prepare ceph-node1:/data/osd0 ceph-node2:/data/osd1 ceph-node3:/data/osd2
```

激活OSD（使用activate命令）

```shell
[ceph@ceph-1 cluster]$ ceph-deploy osd activate ceph-node1:/data/osd0 ceph-node2:/data/osd1 ceph-node3:/data/osd2
```



**创建OSD**

使用`ceph-deploy osd create --data {device} {ceph-node}`创建OSD并添加到集群中

```shell
[ceph@ceph-1 cluster]$ ceph-deploy osd create --data /dev/vdb ceph-node1
[ceph@ceph-1 cluster]$ ceph-deploy osd create --data /dev/vdb ceph-node2
[ceph@ceph-1 cluster]$ ceph-deploy osd create --data /dev/vdb ceph-node3
```

![添加OSD成功](https://s2.ax1x.com/2020/03/11/8An9fS.png)



ok，至此完工！可以去浏览器仪表盘直观地查看部署后的情况。



### 3.4 集群管理

如果之前部署出了问题，可以用下面的命令予以修正。

检查集群状态

```shell
[ceph@ceph-node1 cluster ~]$ sudo ceph health
```

清除配置

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy purgedata ceph-node1 ceph-node2 ceph-node3
[ceph@ceph-node1 cluster ~]$ ceph-deploy forgetkeys
```

清除ceph包

```shell
[ceph@ceph-node1 cluster ~]$ ceph-deploy purge ceph-node1 ceph-node2 ceph-node3
```



---

## 引用

[Ceph分布式存储工作原理 及 部署介绍-散尽浮华](https://www.cnblogs.com/kevingrace/p/8387999.html)

[Ceph-deploy快速部署Ceph分布式存储](https://www.cnblogs.com/kevingrace/p/9141432.html)

[Ceph学习之路-烟雨浮华](https://www.cnblogs.com/linuxk/category/1267134.html)

[全网最详细的Ceph14.2.5集群部署及配置文件详解-PassZhang](https://www.cnblogs.com/passzhang/p/12151835.html)

[Centos7.6安装Ceph（luminous）-会飞的冬瓜](https://blog.51cto.com/6854290/2361910)
