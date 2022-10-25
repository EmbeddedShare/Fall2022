## Near-Data Processing

- **In-Storage Computing：** 在存储设备内完成数据加速处理。
- **Near-Storage Computing：** NSC 技术通常在存储设备（例如 SSD）和主机 CPU 之间的路径上插入一个片上系统 (SoC) 或 FPGA，以加速数据处理，从而减轻主机 CPU 的负担。
- **In-Memory Computing：** 在内存上完成数据加速处理。

## 数据库相关

**NASCENT: Near-Storage Acceleration of Database Sort on SmartSSD**

- FPGA'21

- 使用SmartSSD加速数据库排序操作
- SmartSSD

**Caribou: intelligent distributed storage**

- VLDB'17
- 将部分数据库引擎的计算卸载到存储节点
- near-data processing 但好像使用DRAM
- Xilinx VC709

**POLARDB Meets Computational Storage: Efficiently Support Analytical Workloads in Cloud-Native Relational Database.** 

- FAST'20
- 将表扫描之类的关系数据库操作下推到存储节点
- 使用FPGA(KU15p)实现了flash control

**YourSQL: a high-performance database system leveraging in-storage computing**

- VLDB'16
- 将query的数据扫描卸载到用户可编程的SSD实现数据过滤
- 与Biscuit相同的SSD，在SSD的ARM内核集成了ISC功能

**Query processing on smart SSDs: opportunities and challenges**

- SIGMOD'13
- 卸载部分查询操作
- SmartSSD

**Ibex: an intelligent storage engine with support for advanced SQL offloading**

- VLDB'14
- 一个智能存储引擎，支持卸载复杂的查询运算符。
- NSC

**Less watts, more performance: an intelligent storage engine for data appliances**

- SIGMOD'13
- 与Ibex相关，从查询引擎中卸载查询
- PC通过网口与FPGA通信，SSD通过SATA接入FPGA

**Near-Data Processing in Database Systems on Native Computational Storage under HTAP Workloads**

- VLDB'22
- 利用NDP优化HTAP
- Xilinx Alveo U280 FPGA board

## 通用架构

**Biscuit: a framework for near-data processing of big data workloads**

- ISCA'16
- 一个通用的近数据处理框架
- 在SSD的ARM内核集成了ISC功能

**Willow: a user-programmable SSD**

- OSDI'14
- 实现了通用的SSD应用编程模型
- CSD

**INSIDER: Designing In-Storage Computing System for Emerging High-Performance Drive.**

- ATC'19
- 提供通用的ISC编程模型
- 部署在FPGA，使用DRAM模拟Flash

**Summarizer: trading communication with computing near storage**

- MICRO'17
- 设计了一组编程接口实现任务卸载
- 使用SSD controller的arm核实现ISC

**BlueDBM: An appliance for big data analytics.**

- ISCA'15
- 实现适用于大数据的通用ISC处理引擎
- ISC

## 图相关

**GraFboost: using accelerated flash storage for external graph analytics**

- ISCA'18
- 加速多 TB 图形的外部分析
- Xilinx VC707 (1GB DRAM, 1TB Flash) 

**Hardware/Software Co-Programmable Framework for Computational SSDs to Accelerate Deep Learning Service on Large-Scale Graphs**

- FAST'22
- 加速GNN的推理，支持用户实现GNN算法
- NSC

**ExtraV: boosting graph processing near storage with a coherent accelerator**

- VLDB'17
- 一个用于近存储图处理的框架
- ADM-PCIE-KU3,一个包含FPGA和SSD的开发板。

## 其他

**Analyzing and Modeling In-Storage Computing Workloads On EISC — An FPGA-Based System-Level Emulation Platform**

- ICCAD'19
- 使用仿真系统建立了ISC受益的分析模型
- EISC，开源ISC仿真系统

**DeepStore: In-Storage Acceleration for Intelligent Queries**

- MICRO'19

- 一种用于DNN智能查询的存储加速器架构
- 模拟器

**IceClave: A Trusted Execution Environment for In-Storage Computing**

- MICRO'21
- csd安全方面的研究，在闪存控制器中开发了一种轻量级的数据加密/解密机制来保护从闪存芯片加载的数据。
- 模拟器

**Morpheus: creating application objects efficiently for heterogeneous computing**

- ISCA'16
- 加速对象反序列化操作
- 采用商用SSD和Microsemi的SSD控制器

**In-Storage Computing for Hadoop MapReduce Framework: Challenges and Possibilities**

- TC, 2016
- 将 Map 任务从主机 MapReduce 系统卸载到 ISC SSD
- 扩展SSD固件使用SSD arm核实现

**WAL-SSD: Address Remapping-Based Write-Ahead-Logging Solid-State Disks**

- TC, 2020
- 在SSD中处理事务数据
- 修改三星SSD固件

**SSD In-Storage Computing for Search Engines**

- TC, 2016
- 卸载了5种常用的搜索引擎操作：交集、排名交集、排名并集、差异和排名差异
- SmartSSD

**Two Reconfigurable NDP Servers: Understanding the Impact of Near-Data Processing on Data Center Applications**

- TOS, 2021

- 提出了两种基于NDP的可重构服务器RANS(**R**econfigurable **A**RM-based **N**DP **S**erver)和RFNS(**R**econfigurable **F**PGA-based **N**DP **S**erver)

- NSC

- |                             Name                             | Type/NDPEs  |      Accelerator       | Reconfigurability |
  | :----------------------------------------------------------: | :---------: | :--------------------: | :---------------: |
  | iSSD [[10](https://dl.acm.org/doi/full/10.1145/3460201#Bib0010)] |     ISC     |  General-purpose CPU   |        No         |
  | Active Flash [[40](https://dl.acm.org/doi/full/10.1145/3460201#Bib0040)] |     ISC     |     ARM processor      |        No         |
  | Smart SSD [[16](https://dl.acm.org/doi/full/10.1145/3460201#Bib0016), [33](https://dl.acm.org/doi/full/10.1145/3460201#Bib0033)] |     ISC     |     ARM processor      |        No         |
  | Caribou [[24](https://dl.acm.org/doi/full/10.1145/3460201#Bib0024)] |    NSC/5    |          FPGA          |        No         |
  | BlueDBM [[26](https://dl.acm.org/doi/full/10.1145/3460201#Bib0026)] |     ISC     |          FPGA          |        No         |
  | Biscuit [[20](https://dl.acm.org/doi/full/10.1145/3460201#Bib0020)] |     ISC     |     ARM processor      |        No         |
  | Summarizer [[28](https://dl.acm.org/doi/full/10.1145/3460201#Bib0028)] |     ISC     |     ARM processor      |        No         |
  | YourSQL [[25](https://dl.acm.org/doi/full/10.1145/3460201#Bib0025)] |     ISC     |     ARM processor      |        No         |
  | Ibex [[42](https://dl.acm.org/doi/full/10.1145/3460201#Bib0042)] |    NSC/1    |          FPGA          |        No         |
  | Netezza [[13](https://dl.acm.org/doi/full/10.1145/3460201#Bib0013)] |    NSC/1    |          FPGA          |        No         |
  | Firebox [[3](https://dl.acm.org/doi/full/10.1145/3460201#Bib0003)] | NSC/unknown | System-on-a-chip (SoC) |        No         |
  | NSC [[47](https://dl.acm.org/doi/full/10.1145/3460201#Bib0047)] |    NSC/1    |          FPGA          |        No         |
  |                   RNDP servers [this work]                   |    NSC/n    | ARM processor and FPGA |        Yes        |

