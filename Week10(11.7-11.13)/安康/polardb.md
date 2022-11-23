### **POLARDB Meets Computational Storage: Efficiently Support Analytical Workloads in Cloud-Native Relational Database**

- Wei Cao||Alibaba Group||FAST２０２０

**一、Conclusion**

主要实现计算与存储分离，将数据密集访问任务从计算端卸载到存储端，有效降低计算节点和存储节点之间的运输消耗，POLARDB将scan table从CPU卸载到CSD上。

**二、Background&Motivation**

<img src="https://raw.githubusercontent.com/Ankang0429/images/main/202211221632390.png" alt="image-20221122163209326"  />

本文采用的是分布式异构的体系架构，将table scan分给每一个存储节点

figure(a)--Centralized Heterogeneous:





基于FPGA/GPU的PCIe卡的形式实现,有两个缺点

(1)高数据流量：所有行存储格式的原始数据都必须从存储设备提取到基于FPGA/GPU的PCIe卡中

(2)数据处理热点：数据处理热点：每个存储节点包含大量NVMe SSD

figure(b)--Distributed Heterogeneous:

通过将表扫描直接分配到每个存储驱动器，我们可以消除PCIe/DRAM通道上的高数据流量，并消除系统中的数据处理热点。

三、

