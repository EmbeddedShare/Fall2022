## BlueDBM: An Appliance for Big Data Analytics

Sang-Woo Jun/ISCA ’15/麻省理工

#### 总结

该篇论文的启发点似乎来自于RAMCloud，随机访问造成复杂的数据查询进程缓慢，而且DRAM容量较小，一种方法可以增加RAMCloud中DRAM数量。本文提出了另一种方法，采用分布式Flash存储的单个基架Blue DBM，Flash存储比磁盘有更好的随机访问性能。

Blue DBM主要有以下Capabilities：

1. A 20-node system with large enough flash storage to host
Big Data workloads up to 20 TBs;
2. Near-uniform latency access into a network of storage de-
vices that form a global address space;
3. Capacity to implement user-defined in-store processing
engines;
4. Flash card design which exposes an interface to make
application-specific optimizations in flash accesses.

#### 一、系统架构

![image-20221129165550243](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129165550243.png)![image-20221129171111816](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171111816.png)

每一个BlueDBM存储设备通过PCIe接口嵌入到host server中，它构成了Flash存储，一个in-store 处理引擎，多个高速网络接口以及一个on-board DRAM。Host Servers通过以太网或其他通用网络连接在一起。主机服务器能通过基于PCIe的主机接口来访问BlueDBM存储设备。内置（in-store）处理器能够执行数据计算。In-store处理器可以访问四个主要部件：Flash接口，网络接口，主机接口和on-storage DRAM 缓冲

本文实现了一个带有标签重命名的Flash Interface Splitter来管理多个用户，如下图3所示

![image-20221129171150303](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171150303.png)

如下图4所示，呈现了一个网络结构。交换机有两层，分别是内部交换机和外部交换机。

![image-20221129171201041](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171201041.png)

图7描述了Flash读操作的主机接口结构

![image-20221129171535095](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171535095.png)

#### 二、软件接口

![image-20221129171610611](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171610611.png)

软件结构如图8所示，有三个接口可供用户程序使用，分别是

（1）文件系统接口（2）块设备接口；（3）加速接口

RFS实现了FTL的某些功能，包括逻辑到物理地址映射以及垃圾回收机制。这个能以更低的内存需求而获得更好的垃圾回收效率。在BlueDBM中的文件系统接口就是采用了RFS同样的范式。为了更有效的共享硬件资源，BlueDBM运行一个调度器（scheduler）分配可用的硬件加速单元来完成用户应用。

#### 三、硬件实现

![image-20221129171722908](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171722908.png)

如图9所示是BlueDBM的硬件实现。作者们使用了现场可编程门阵列（FPGA）来实现该In-store处理器，该架构还包括Flash、host和network控制器。然而，BlueDBM架构不应该局限于基于FPGA的实现，这里BlueDBM是通过高级硬件描述语言Bluespec来开发的。该集群由20rack-mounted Xeon服务器组成，每个服务器有24个核并且有50GB的DRAM。每一个服务器有一个Xilinx VC707 FPGA开发板通过PCIe接口连接。Host 服务器基于UBuntu版本的Linux。



![image-20221129171759708](C:\Users\AnKang\AppData\Roaming\Typora\typora-user-images\image-20221129171759708.png)

如图10所示是一个单节点的各部件组成。众多服务器中的一个有512GB的Samsung M.2 PCIe SSD用于性能比较。Connectal的PCIe实现限制1.6GB/s读和1GB/s写。