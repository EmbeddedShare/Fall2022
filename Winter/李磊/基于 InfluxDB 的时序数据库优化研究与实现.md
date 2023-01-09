# 基于 InfluxDB 的时序数据库优化研究与实现

## 1. 背景
1. 随着近年来各类互联网服务的持续增长，各类存储系统和计算系统需要在愈来愈大量的机器上部署。对这样大规模的机器、系统和服务进行监控已成一大挑战。---关键需求包括有效地管理这些监控数据并提供实时查询服务。
2. 传统数据库管理系统 (DBMS, Database Management System) 在处理时序数据存在许多不足，比如难以存储如此大规模时序数据，为了存储和分析大规模的时序数据，业界提出了各类以时序数据库 (TSDB,Time Series Database) 为核心的时序数据管理系统 (TSMS, Time Series Management System)。
3. 已有的一些TSMS：
- BTrDB，将数据组织在内存中，批量地写入后备存储设备。它在内存中对不同规模的统计数据进行预先计算，从而减少后续的磁盘I/O。但BTrDB 不支持多维时序数据模型。
- Akumuli ，使用一种名为 Numberical B+-tree 的存储结构，在某种程度上接近于 BTrDB 的设计。Akumuli 加入了多维数据模型和索引来支持多维查询。
- Gorilla ，是一个 Facebook 开发的针对大型监控系统的后备持久化存储设备的 write-through 缓存系统。它提出了一种高效的时序数据压缩算法，高效的数据压缩和简化的数据结构允许 Gorilla 能够存储内存中的全部数据。然而 Gorilla 的数据模型并不支持多维查询。
- Prometheus ，为监控系统而构建的时序数据库。思想类似于Gorilla，即使用内存和本地持久化存储设备提供高效的数据摄取和查询。为了降低内存使用，Prometheus 将数据按照时间划分到多个块 (blocks)，只把最新的块保存在内存中，尽管如此它还是需要大量内存来维护动态的索引结构，而且没有对元数据进行压缩。
- TimescaleDB ， 基于 PostgresSQL [11] 的时序数据存储引擎，它在内存中创建一张表来保存最近插入的数据，当这张表在内存中的驻留时长达到阈值或表的大小达到阈值就将其刷到磁盘上。TimescaleDB 用来保存时序数据的若干表彼此完全独立，拥有各自的元数据和索引。由于缺乏有效的数据压缩算法，TimescaleDB 的性能受限于内存的高效利用。

## 2. 动机
1. InfluxDB 的时序数据存储引擎--Time-Structured Merge (TSM) Tree，衍生自 LSM-Tree，在写性能上存在写停顿、写放大等痛点。
- 写放大：存储引擎为了保证多个SSTable 之间关键字的有序性而进行大量数据的归并重写 (以下简称 “合并”)，使得磁盘数据的写入量多于用户提交的写入量。
- 写停顿：由于后台任务所调度的大量 SSTable 的合并占据了磁盘带宽，导致 Immutable Memtable 无法在短时间内完成刷盘，新的 Memtable 亦无法及时创建，使得用户提交写请求后出现短时间内无法写入的现象。
2. InfluxDB 已经拥有的分布式集群方案中，元节点集群实现了分布式 CAP (一致性、可用性、分区容错性) 中的 CP。数据节点集群实现了 AP，可用性优先，不保证数据的强一致。
![在这里插入图片描述](https://img-blog.csdnimg.cn/9d97a6190e8f4a5a85a4146d9d21c2e4.png)
但数据节点集群的一致性级别及其 “无主多写” 的方案并不能适应业界对于分布式数据库的高可靠性要求。

## 3. 系统设计
#### 3.1 存储引擎TSM改造
约束条件：时序数据库写入的数据都是带时戳的，来自 IoT 设备的同一数据源的时序数据的时戳应该是单调递增的。
如果不让存储引擎无条件支持乱序数据的写入，仅在小范围时间区间内允许乱序，就可以简化InfluxDB 的 TSM 合并逻辑，进而降低写放大，消除写停顿，以及 TSM 合并期间的内存高峰。
所以，在 InfluxDB 中引入一个乱序表(DisorderTable)，存放时戳紊乱的数据。
进一步，由于 TSM 文件内同数据源数据的时戳是单调递增的，TSM 合并也就没有太大的意义了，那么就可以将 TSM 文件的层数降下去。考虑到本论文并不是绝对不允许时序数据的乱序写入，所以将
乱序表里的数据与 Immutable Memtable 刷盘后的 L1 TSM 里的数据合并，产生新的 L2 TSM。这样一来，TSM 的层数就可以降到 2 层。此外，对于乱序表里的数据如果不能和L1合并，则认为它是迟到时间过长的数据，直接丢弃。
#### 3.2 乱序表：
![在这里插入图片描述](https://img-blog.csdnimg.cn/aa7b830a91584803a66a62b406695d4c.png)
定义两个数据结构来记录每个数据源下数据的时戳范围 (即区间的左右端点)：seriesKey2MinTime
和 seriesKey2MaxTime。
在每次写入成功后更新 seriesKey2MaxTime，对于时戳下界 seriesKey2MinTime，在 Memtable 转变为 ImmutableMemtable 并刷盘后，新的 Memtable 被创建出来，此时用当前 seriesKey2MaxTime覆盖 seriesKey2MinTime 即可。图4-2描述了时序数据在 L0 的写入流程。
![在这里插入图片描述](https://img-blog.csdnimg.cn/bff498eecd9647a483e6d2902ea6774e.png)
1. 乱序表的持久化：
当 Memtable 转变成 Immutable Memtable 之后，一个新的 WAL 会被创建出来，随即将乱序表刷到这个 WAL 的开头，后续的写请求到来时相应的日志则继续往后追加。此外，还需要将 Memtable 里的 seriesKey2MinTime 和 seriesKey2MaxTime持久化，保证系统重启后相关逻辑不出错
WAL 日志项的写入流程如图4-3所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/b38ac3d23a7d4142a1ddb692c99aa7ca.png)
2. 乱序表的合并
L1 → L2 的 TSM 层间拷贝其实是在基于对同数据源时序数据的单调递增约束下的弱化的 TSM 合并操作。当 L1 的 TSM 文件个数达到阈值从而需要刷到 L2 时，将当前乱序表里的数据点和 WAL 里属于上一个乱序表内的数据点合并。
乱序表合并的流程可大致用算法4-2描述。
![在这里插入图片描述](https://img-blog.csdnimg.cn/89dbfbbf3bdc4a1ea8a561845335ea5d.png)

#### 3.3 分布式架构设计
系统架构如图所示，主要分为计算层和存储层。计算层由两个 InfluxDB 数据节点集群和一个元节点集群构成，均使用 Raft 算法保证集群内节点数据的一致性，并提供容错能力。
[Raft算法简介](https://cloud.tencent.com/developer/article/1101009#:~:text=Raft%E6%98%AF%E4%B8%8D%E4%BE%9D%E8%B5%96,eader%E7%BB%93%E6%9D%9F%E3%80%82)
![在这里插入图片描述](https://img-blog.csdnimg.cn/59760fd37c6d42359c28926f37b50a29.png)

组成数据节点集群的 InfluxDB 存储引擎经过改造，将原本的 4 层 TSM 合并操作弱化为 2 层。L2 TSM 作为冷数据而被存储到远端存储池里的 StoreNode 上面。远端存储池里的 StoreNode 的状态信息由一个同样运行 Raft 算法的 master 集群进行维护。
整个系统通过 influx proxy 将读写接口暴露给用户。influx proxy 提供 “负载均衡、分库分表” 的功能，将用户的读写请求根据 database 和 measurement 转发给相应的 InfluxDB 数据节点集群。
对于查询请求，L2 TSM的访问需要由数据节点向StoreNode 发送一个远程查询请求，将其返回
的数据和本地数据合并后一起返回给用户。
按照 raft 算法的基础流程，在一个副本集中处理一次客户端的读写请求大致要经历以下步骤：
(1) leader 收到来自客户端的请求；
(2) leader 把请求封装为日志项追加到本地日志中；
(3) leader 将该日志项发送给 followers，followers 将收到的日志项追加到本地日
志中；
(4) leader 等待 followers 的回复。如果大多数 followers 回复成功，leader 就提交
该日志项，并将其应用到上层状态机；
(5) leader 向客户端确认本次请求。
对于读请求如果每一次都这么完整走一遍流程比较繁琐，所以读性能有所降低。针对此，提出了两种优化方案：
 - read index，当 leader 确认读请求的同时记录当前的 commit index 作为本地读索引 (read index)，然后向 followers 发送心跳以确认自身的 leader 身份。如果自己仍是 leader，那么只要它的 applied index 大于 read index 就可以在后续收到客户端的读请求时直接从本地读取数据并返回，无需走一边 raft 的日志同步流程。
 - follower read 让 followers 也能响应客户端的读请求。当 follower 收到读请求后，它向 leader 查询当前最新的 read index。如果 follower 的上一轮的 applied index 大于等于 leader 的 read index，就允许把直接在本地读取数据返回给客户端，否则要等待日志被提交到 read index 的位置。

###### 计算层
1. 元节点
元节点集群使用 Raft 保证一致性。针对元数据的读写请求都会在元节点集群内进行线性一致性排序。raftmeta.MetaService 实现了元节点的核心逻辑，类图如下。
![在这里插入图片描述](https://img-blog.csdnimg.cn/cf446a23ce8d41c1a935b69260c30b2c.png)
raftmeta.MetaService 接收到来自 meta.Client 的元数据读写请求后，会将请求参数封装为 Raft 日志项，通过 ProposalAndWait() 将日志项打到 Raft 层进行同步。当携带请求参数的 Raft 日志项在大多数节点之间达成一致后确认提交，ProposalAndWait() 的阻塞等待被解除，这也意味着完成了一
次元数据的读写操作。

2. 数据节点 
数据节点依靠 coordinator.ClusterMetaClient 向元节点读写元数据，如图5-4所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0f70a913bf294c37bc968e7de3715cd2.png)
3. Raft 模块
Raft 模块的流程以 (*RaftNode).ProposalAndWait() 为起点，主要步骤如图5-5所示，
![在这里插入图片描述](https://img-blog.csdnimg.cn/996146f9485a47d1a6cd8c670b698c53.png)
①将 Raft 日志 (即一条提议) 打到 Raft状态机，按照 Raft 算法流程完成日志的同步和提交，
②等待提议被应用到集群内大多数节点上。
③Raft 状态机将已提交的日志项通过一个通道传递给 (*RaftNode).processApplyCh，这里面就包含之前那条提议。
④(*RaftNode).processApplyCh将提议传递给 (*RaftNode).applyCommitted，从而将其中的操作应用到上层状态机。
⑤待提议被应用后将结果返回给 (*RaftNode).ProposalAndWait，解除它对这条提议的阻塞
⑥最后将结果反馈给用户。

###### 存储层
1. StoreNode内部结构
StoreNode 内部实现如图5-11所示，由一个 slave 进程和一个 InfluxDB 服务端进程组成。slave 负责接收计算层的数据节点发送过来的 L2 TSM 并将其导入到InfluxDB。
![在这里插入图片描述](https://img-blog.csdnimg.cn/3a9ad60e647d4495b3af2ed976a66fcf.png)
slave端的四条协程：
- 周期性地向 master 发送心跳包
- 在 TCP 端口上监听，接收来自计算层数据节点或其他同级 slave 的L2 TSM文件
- 在 TCP 端口上监听，接收 master 发送的文件拷贝命令
- 将 L2 TSM 文件导入到与 slave 绑定的 InfluxDB 服务端，以便计算层数据节点完成远程查询。

为了保证存储层的高可靠和高可用性，需要使 StoreNode 具备良好的容错能力。定义两类 StoreNode：active StoreNode 和 backup StoreNode。active StoreNode指的是被分配出去承担了 L2 TSM 存储任务的 StoreNode，而未被分配出去的是 backup StoreNode，作为冗余后备。如图5-12所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/493bbc90f659441aaba4b8705af28216.png)

2.  Master集群
Master 集群利用 Raft 提供容错能力，并利用 Raft 将任意节点上接收到的StoreNode 心跳包在集群内同步。Master 作为存储池的管理者，需要提供两个 RPC （远程程序调用）供计算层的数据节点调用：
- AssignSlave 分配一个存储节点给数据节点作为镜像，用于保存 L2 TSM 文件。请求参数与返回值定义如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/0ebe8201d8d940348a61ce94cc6b73d7.png)

- GetRemoteNodeAddr 数据节点执行远程查询时获取镜像存储节点的网络地址。请求参数与返回值定义如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/8fbf422c9d474443849cfd250f8dac23.png)
所以，Master 的主要功能包括：（1）接受 slave 的心跳包，更新 slave 的状态信息；（2）根据心跳超时发现宕机的 slave，调度后备 slave 替换之；（3）为计算层的数据节点提供 RPC 接口。

## 4. 实验
![在这里插入图片描述](https://img-blog.csdnimg.cn/e01de0d9b9bf49d0b5b2753aece46b1b.png)
其中，Kapcitor 和 Chronograf 是 InfluxDB 生态下的可视化数据分析工具，用于在写入测试中统计 InfluxDB 服务端的吞吐量。
1. 乱序数据写入测试
![在这里插入图片描述](https://img-blog.csdnimg.cn/b3d43f6c6efb41cebce4f2347525381c.png)
四组测试数据在写入过程中的 Δ-InfluxDB 的吞吐量曲线如图6-2所示，由于乱序数据占比很低，所以吞吐量曲线较为平稳，没有出现剧烈的抖动。
![在这里插入图片描述](https://img-blog.csdnimg.cn/85001b3bf9b9493aa4cc977f0b081eaa.png)
最后统计四组测试下最终写入的数据量以及被丢弃的乱序记录条数及其在乱序记录中所占比例，如表6-3所示。显然，当存在乱序数据时 Δ-InfluxDB 不能保证将其全部写入，少量乱序数据因其时戳落后于 L1 的全体 TSM 文件，所以被丢弃了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f5cf65a4962e41e2acc9d5e74823b5e0.png)

2. 压力写入测试
图6-3(a)和图6-3(b)分别对应原生 InfluxDB 和 Δ-InfluxDB。图6-3(a)中那些吞吐量曲线几乎将至 0 的位置就是因为发生了写停顿。Δ-InfluxDB 极大地弱化了 TSM 的合并操作，所以图6-3(b)的吞吐量曲线虽然存在轻微抖动但未出现降至 0 的写停顿情形。
![在这里插入图片描述](https://img-blog.csdnimg.cn/5b7fad2f8786463e9ac3591fb7b96ddb.png)
在进行写入测试过程中监控 InfluxDB 服务端进程的内存占用，如图6-4所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/c808a59bd04d4eef8bde16effb8f90ad.png)
Δ-InfluxDB 因为弱化了 TSM 合并，而 L1 → L2 的 TSM 文件拷贝并不会成为内存瓶颈，所以全程未出现内存高峰。

3. 分布式系统功能测试
测试了无节点宕机、存储层全部 active StoreNode 宕机、大量节点（测试存储层全部 StoreNode 宕机、master 集群、计算层元节点和数据节点集群均有一个节点）宕机的情况下系统性能：
测试结果显示系统均能正常运行，说明存储层和计算层有较高的容错能力。

## 5.总结
当今对IOT设备的需求大，其产生的数据需要存储和分析，对此时序数据库的应用研究显得十分重要。
由于 InfluxDB 的时序数据存储引擎——TSM 存储引擎衍生自 LSM-Tree，所以 TSM 存储引擎存在写性能上存在写放大和写停顿的问题。本文对 InfluxDB的 TSM 存储引擎的优化目标就是消除写停顿和降低写放大，同时降低持续写入过程中的内存资源开销，从而满足 IoT 应用场景对时序数据库的的性能需求。另外，InfluxDB 原生的集群方案在如今业界对分布式系统的设计要求下来看有些差强人意，
本文提出新的 InfluxDB 分布式架构采用了 “存储-计算” 分离的设计思路，通过 Raft算法保证副本集的强一致性和分区容错性，符合业界主流。对 InfluxDB 分布式架构的功能测试达到预期结果，整个系统具备良好的可用性、可靠性以及水平扩展能力。
 

