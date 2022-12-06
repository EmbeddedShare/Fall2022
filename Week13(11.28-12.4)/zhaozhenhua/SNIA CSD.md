## Computational Storage Architecture and Programming Model

[TOC]

### 1 研究范围

本SNIA文档定义了对支持计算存储的硬件和软件的推荐行为。规范侧重于定义能够跨计算存储设备 (CSxes)（例如计算存储处理器、计算存储驱动器和计算存储阵列）与主机代理或其他 CSxes 之间的接口实现的功能和操作。

相关概念：

- **Management：**在安全策略的基础上主机代理允许执行的操作：
  - **Discovery：**定义和决定可计算存储资源（CSR）的机制；
  - **Configuration：**通过编程参数实现初始化，执行，以及资源分配；
- **Security：**与CSxes相关的安全性考量；
- **Usage：**允许主机代理或者CSx将可计算存储任务卸载到CSx中，包括为目标CSx提供存储在CSx本地或者其他非本地CSx上数据的位置信息；

文档对主机代理和CSx(s)之间接口的物理性质不做限制，上述概念可以通过不同物理接口实现。文档对主机代理以及CSx(s)的存储协议也不做限制，但是列举了一些可能支持的存储协议：

- **Logical Block Address：**数据被组织成固定大小的逻辑单元，操作在该逻辑单元上是原子的。数据通过数值索引映射到逻辑块地址空间；
- **Key-value:**数据不是固定大小的，通过key来索引；
- **Persistent Memory:**可按字节寻址的非易失性存储；

规范定义了通过多个可计算存储函数传递数据的操作，这些函数不一定在同一个CSx上。此外规范还定义了用来请求多个可计算存储函数执行一系列任务的操作。

### 2 定义与缩写

- **Allocated Function Data Memory：**

  为特定CSF实例分配的函数数据内存（FDM）；

- **Computational Storage：**

  提供存储耦合计算、主机端计算卸载或者减少数据搬运的架构；

- **Computational Storage Array (CSA)：**

  包含一个或多个CSEs的存储阵列；

- **Computational Storage Device (CSx)：**

  包括可计算存储驱动器、处理器、阵列；

- **Computational Storage Drive (CSD)：**

  包含一个或多个CSEs以及持久数据存储的存储元素；

- **Computational Storage Engine (CSE)：**

  可以执行CSFs的组件，例如CPU、FPGA；

- **Computational Storage Engine Environment (CSEE)：**

  CSE的操作环境，例如操作系统、容器、eBPF、FPGA字节流；

- **Computational Storage Function (CSF)：**

  通过CSE部署并执行的一系列特定的操作，例如压缩、RAID、erasure coding、正则表达式、加密算法；

- **Computational Storage Processor (CSP)：**

  与某个存储系统相关的包含一个或多个CSEs的组件，不提供持久存储；

- **Computational Storage Resource (CSR)：**

  CSx所需的用来执行计算的资源，例如 FDM, Resource Repository, CPU, memory, FPGA resources；

- **Function Data Memory (FDM)：**

  用来存储计算存储函数所涉数据的设备内存，包括已分配的和未分配的函数数据内存；

- **platform firmware：**

  平台上所有的设备固件的集合；

### 3 可计算存储设备的操作概念

#### 3.1 Overview

| ![image-20221202105320724](https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202105320724.png) | ![image-20221202105348729](https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202105348729.png) | ![image-20221202105407930](https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202105407930.png) |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |

CSx包含如下组件：

- 一个资源库CSR包含：
  - 计算存储函数
  - 计算存储引擎执行环境
  - 函数数据内存(FDM)，可以被划分为已分配函数数据内存(AFDM)以及一个或多个可计算存储引擎(CSEs)
- CSD控制器或者CSA的阵列控制器；
- 设备内存；
- CSD或CSA的持久存储；

CSR是CSx所需的用来存储和执行CSF的资源；CSE本质上也是CSR，可以被编程以提供一个或多个CSF；CSEE是CSE的操作环境；CSF由CSEE中的CSE部署和执行，用来执行用户定义的一系列操作；激活指将CSEE与CSE关联或者将CSF与CSEE关联的过程。CSEE和CSF的激活都需要分配所需的资源。当CSEE与CSE失去关联或者CSF与CSEE失去关联的时候（类似于函数执行完成或者操作环境不再使用）则需要将对应的CSEE或者CSF停用并释放已分配的资源。

CSE需要先拥有已激活的CSEE才能够激活CSF，CSE拥有与之相关的FDM。CSE在被生产时可拥有一个或多个激活的CSEEs和CSFs，这些资源可以被主机端通过管理系统或者IO接口调用。CSE的CSEEs和CSFs也可以是从主机端下载并激活的。CSE在被生产时可能带有不可更改的CSFs。CSEE可以预先安装也可以从主机端下载，其需要被激活使用。功能数据内存（FDM）是一种可用于CSF，用于作为CSF操作一部分使用或生成的数据的设备内存。

CSD（computational storage drive）可以执行一个或多个CSFs并提供持久存储，CSD包含一个存储控制器，CSRs，设备内存以及持久数据存储。

#### 3.2 Discovery

与特定系统相关，操作涉及CSD架构之外。例如当CSx设备被识别，其中的相关信息需要被识别。涉及到识别CSx上的CSR，CSR识别返回所有可用资源，例如CSEs，CSEEs，CSFs以及FDM。CSE的识别过程还包括识别CSEEs内部已被激活的CSEEs以及CSFs。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202105235358.png" alt="image-20221202105235358" style="zoom: 80%;" />

上图是在CSx上部署运行CSF的流程图，大体就是查看所需的组件有没有被激活。首先查看需要的CSF是否在CSEE中已被激活，如已激活，再看是否需要部署。否则再查看CSEE是否属于激活状态，如果不是进行下一步操作。。。。大体就是逐步检查所需要的资源。

#### 3.3 Configuration

CSE，CSEE的部署可以再特定CSF调用之前执行也可以作为参数随着CSF调用执行。

#### 3.4 Security

安全性考量基于以下假设：

- 系统由单个物理主机或虚拟主机，以及单个或多个CSxes组成；
- 主机负责CSxes运行生态系统的安全性；
- CSx的安全性要求和SSDs/HDDs常见的安全需求相媲美；

在多主机情况下安全性考虑更加复杂。

**CSx Security Considerations**

- **CSx Sanitization**

  CSF的所有实例应该被终止，所有活跃的CSEEs和CSFs应该失效，FDM涉及的内存分配以及FDM应该被清理。

- **CSx Data at-rest Encryption**

  存储在物理介质上数据的安全性经常被人忽视，如果未加密的数据库文件被未授权的用户获得，用户可以完整还原数据库中的数据。一般静态数据加密的方式有全盘加密和数据库透明加密（TDE）。全盘加密是将磁盘上所有的数据进行加密，它在写入和读取磁盘时对数据进行加解密，微软的Bitlocker就是其中一种，它能够很好的应对物理磁盘被窃取的情况，但是如果数据被从磁盘上拷走，那么拷贝出去的数据是处于解密状态。另一种是则是数据库透明加密，这种加密方式在写入和读取数据库的时候会进行相应的加解密操作，所以即使数据库文件被窃取，只要主密钥还在，就不会造成数据丢失。CSx需要提供与其他存储设备类似的静态数据加密功能。

- **CSx Key Management**

  与静态数据加密相关的密钥管理。

- **CSx Storage Sanitization**

  对于被用户删除的数据，CSx可以拥有对该数据的清洗能力。有clear和purge两种方式。cryptographic erase保证所有用来加密数据的密钥的副本全部被清洗，并对清洗操作的结果进行验证。

- **CSx Roots of Trust (RoT)**

  信任根以高度可靠的硬件和软件组件的形式存在，它们执行特定的、关键的安全功能，为构建安全提供了坚实的基础，可用于支持静态数据加密、密钥管理、以及认证功能。

- **CSx Software Security**

  CSx的软件安全主要涉及CSEE和CSFs。软件可能是预先安装的或者从主机端下载的。CSx需要提供以下软件安全机制：
  
  - 下载CSx软件的完整性检验，例如利用checksums去检查错误；
  - 验证代码来自特定的来源，并且代码没有被第三方更改或破坏，例如使用代码签名；

#### 3.5 CSF Usage Overview

host可以通过两种方式使用部署的CSF：direct usage model、indirect usage model；**直接使用模型中**，主机端发送计算请求（computation request），指定一个CSF对FDM中的数据进行操作。主机端或者存储端与FDM的数据流动可以在CSF之外完成；**间接使用模型中**，主机向存储控制器发送一个存储请求，CSF可以基于以下信息对存储请求相关的数据进行操作：存储请求中的参数、数据地址、其他数据特征（例如数据大小）。

- **Direct CSF Usage Model：**

  主机端发送指令调用CSF，CSE在AFDM中的数据上执行所需的计算，并将结果存放到AFDM中，CSE向主机端发送响应。

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202215950432.png" alt="image-20221202215950432" style="zoom: 67%;" />

- **Indirect CSF Usage Model：**

  下图假设读取操作对所读数据进行计算。

  - 主机端配置CSD，将特定的CSF与指定的读操作联系起来。
  - 主机端发送存储请求给存储控制器：存储请求和目标CSF相关联、存储控制器决定具体哪个CSF与存储请求相关联；

- 存储控制器将数据从底层存储介质中移动到AFDM；

- 存储控制器指示CSE在FDM对应的数据上执行所需的计算；

- CSE对数据进行计算，如果有结果放置在FDM中；

- 如果有计算结果，存储控制器将FDM中的结果返回给主机端；

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221202220022118.png" alt="image-20221202220022118" style="zoom: 67%;" />

### 4 Example Computational Storage Functions

#### 4.1 Compression CSF

从源地址读取数据，对数据进行压缩或者解压缩，然后将数据结果写回目的地址。

CSF的configuration操作将指定压缩算法以及相关参数。

CSF指令指定源地址以及数据长度，目的地址以及最大长度。

#### 4.2 Database Filter CSF

从源地址读取数据，根据谓词条件执行数据库投影操作（列选择）和过滤操作（行选择），将结果写回目的地址。

CSF的configuration指定数据库格式、表结构、选择和过滤谓词、以及其他相关操作。

CSF指令指定源地址以及数据长度，目的地址以及最大长度。

#### 4.3 Encryption CSF

加密CSF从源位置读取数据，对数据进行加密或解密，并将结果写入目标位置。

CSF的configuration指定了加密算法、密钥信息和相关的参数。

CSF指令指定源地址以及数据长度，目的地址以及最大长度。

#### 4.4 Erasure Coding CSF

纠删码CSF从源位置读取数据，对数据执行EC编码或解码，并将结果写入目标位置。

CSF的configuration指定算法以及相关参数。

CSF指令指定源地址以及数据长度，目的地址以及最大长度。

#### 4.5 RegEx CSF

RegEx CSF从源位置读取数据，用正则表达式模式对数据进行匹配或转换，并将结果写入目标位置。

CSF的configuration指定正则表达字符串。

#### 4.6 Scatter-Gather CSF

从源数据地址集合中读取数据，写数据到目标地址集合。

configuration操作没有任何参数。

#### 4.7 Pipeline CSF

管道CSF根据数据流规范对数据执行一系列操作，允许将不同的CSF命令以标准化的方式组合在一起。

configuration操作没有任何参数。

CSF指令指定一个指令集合，指令顺序，依赖关系，定义命令之间地址关系的计算。

#### 4.8 Video Compression CSF

视频压缩CSF从源位置读取数据，压缩或解压缩视频，并将结果写入目标位置。为了容纳多个并行压缩，视频压缩CSF可以支持单个压缩流或多个压缩流。

视频压缩CSF指定流、压缩算法和相关参数。

#### 4.9 Hash/CRC CSF

从源数据地址读取数据，计算Hash/CRC value，将计算结果写回目的地址。

#### 4.10 Data Deduplication CSF

冗余数据删除

#### 4.11 Large Data Set CSFs

###  5 Illustrative Examples

这一部分介绍一些具体的应用案例

#### 5.1 CSFs on a Large Multi-Device Dataset using Ceph

ceph https://blog.csdn.net/weixin_43860781/article/details/109053152

大型多设备数据集跨多服务器，样例使用TCP/IP进行服务器间通信。一个大型数据集将被分割成块，这些块对应用程序具有语义意义，并跨一组存储设备进行存储。数据映射以及CSF被发送到多个设备的功能需要被实现，CSF被发送到对应设备后就可以对数据进行操作。很多系统可以将存储空间拓展到多个设备，例如ceph。Ceph允许许多应用程序在潜在的数千个设备上联合共享被称为对象的数据碎片。Ceph负责在所有设备上映射每个对象的位置。虽然被称为对象存储守护进程（OSDs）的中间服务器使用本地存储互连，但主要的应用程序互连是TCP/IP。应用程序通过一个唯一的键定位并与对象交互，该键转换为一个唯一的IP和TCP端口地址。应用程序并不指定这个地址，而是让Ceph管理对象的位置，从而从实际位置抽象出客户机。

<img src="https://img-blog.csdnimg.cn/20201013162347608.png" alt="img" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221205164059755.png" alt="image-20221205164059755" style="zoom:67%;" />

File Block S3 API都是基于Ceph RADOS API实现的，这个API允许存储和检索任意大小的对象，以及对对象执行方法。结合计算存储架构，Ceph可当作CSA，OSDs可当作计算存储处理器，CSE可以被部署在OSD中以被识别。但是，软件体系结构中没有任何东西可以阻止将Ceph OSD作为单个设备服务器运行。在这种情况下，OSD服务器将被视为一个计算存储驱动器（CSD）。

| <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221205172717370.png" alt="image-20221205172717370" style="zoom: 50%;" /> | <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221205172742485.png" alt="image-20221205172742485" style="zoom:67%;" /> |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

接下来讨论discovery、configuration以及usage。该样例中大部分的discovery操作都是由Ceph完成。Ceph通过对象类的定义，在OSDs中可运行某个对象的代码，该属性可以被视为CSE。对象类定义包含了一些方法，这些特定对象的方法可以在OSD中执行。一个对象类可被视为CSF，预先加载的或者下载的CSFs都可以被创造。发现并识别这些对象类是必要的。应用使用CSF参数调用CSF，CSF是从应用中抽象出来的Ceph对象类。CSF参数包括：对象类的方法、目标对象的名称、以及输入的参数。CSF的调用由Ceph的RADOS rados_exec()函数处理。一旦return，任何输出参数包括调用状态都会被返回。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221205202752878.png" alt="image-20221205202752878" style="zoom: 67%;" />

skyhook是一个开源中间件，可以为PostgreSQL提供Ceph后端服务。其为PostgreSQL中的数据表划分边界，将N行数据存入一个单独的对象中。如果N=1000，数据表有100,000行数据，那么Skyhook将创建100个对象，这些对象会被 分配到Ceph中的100个OSDs/Devices中。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221205204528087.png" alt="image-20221205204528087" style="zoom: 67%;" />

Skyhook在每个Ceph OSD节点（CSE）都定义了自己的对象类（CSF），这个对象类实现了执行SQL查询的方法。当PostgreSQL接收到用户提交的SQL查询，其将查询提交给Skyhook，Skyhook通过并行调用RADOS exec()函数（CSF）将查询提交给属于该表格的100个节点。该调用（CSF）将查询传输给SQL执行方法，每个节点单独评估自己保存的数据并将结果返回给skyhook。skyhook将所有的结果合并成单独结果，最后返回给PostgreSQL。系统的可扩展性和并行性都可得到提升。

#### 5.2 Using a Containerized Application within Linux (CSEE with included CSF)

样例为一个基于NVMe PCIe计算存储驱动器，由一个典型的基于NVM存储设备的逻辑块地址（LBA），多个位于同一个控制器上的可编程应用处理器组成，控制器可以运行Linux操作系统，从而运行容器化的数据搜索应用。样例基于以下假设：

- 服务器运行现代Linux内核以及用户空间分发；
- 单端口单主机系统（不考虑多端口NVMe设备）；
- 不使用虚拟化；
- CSE中预先激活了CSEE（运行Linux OS - CSEE1）；
- 设备存储中的CSEE是一个container（CSEE2），带有CSF1；
- CSF1运行一个AI应用；
- 没有使用对等功能或名称空间；
- 利用现有的PCIe和NVMe方法；

NVMe linux驱动器被稍作修改，要求具有识别和部署CSD的能力。NVMe/PCIe本身就具有健壮的控制器发现和创建机制。PCIe设备（例如NVMe控制器）在上电时可以通过PCIe总线枚举或者在运行时通过热插拔和总线扫描完成。现代Linux操作系统可以通过上述两种方式检测PCIe设备，使得新NVMe控制器可以在系统运行的任何时段被发现。

一旦NVMe控制器被检测，NVMe CSD能够通过NVMe识别管理命令发现控制器的资源以及处理能力，Discovery 流程如下：

- PCIe枚举过程发现CSD的NVMe控制器；
- 可计算存储NVMe Linux驱动器与该PCIe设备绑定；
- 驱动器发现对应CSD的资源，CSE内被激活的CSEE运行Linux操作系统环境（CSEE1）；

| <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221206105538682.png" alt="image-20221206105538682" style="zoom: 50%;" /> | <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221206105613056.png" alt="image-20221206105613056" style="zoom:50%;" /> |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

Configuration流程如下，CSEE1在CSE中已被激活：

- A) docker container的CSEE2被移动到CSD的设备存储中，Admin命令会使应用程序处理器引导到Linux环境中(?)。支持Docker的软件会作为Linux启动过程的一部分自动加载。
- B) 一旦Linux启动，用于Docker环境的CSEE2将随CSF1一起加载。
- 系统将通知计算存储NVMe驱动程序，以便可以下载特定的计算工作负载。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/SINA/image-20221206105644631.png" alt="image-20221206105644631" style="zoom:50%;" />

一旦部署完成并且CSEE1，CSEE2被激活且CSF1准备好处理数据，usage过程可以启动：

- CSF1被主机端告知在数据上执行的函数；
- 输入数据从设备存储加载到设备内存，放置在FDM的AFDM中；
- CSF1对数据进行处理，将结果存放到函数指定的设备存储地址中；
- 主机可根据需要访问CSD中的输入数据和输出数据；

车牌识别应用案例：OpenALPR

#### 5.3 Data Deduplication CSF

重复数据删除是一种用于减少存储在存储设备上的数据量的技术。压缩会导致删除存储块或存储段中的重复字节或流，但重复数据删除会导致删除匹配的存储块或存储段。

<img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20221206112855206.png" alt="image-20221206112855206" style="zoom:67%;" />

#### 5.4 A Computational Storage Function on a NVM Express and OpenCL based Computational Storage Drive Illustrative Example

#### 5.5 Data Compression CSF Example

#### 5.6 Data Filter CSF Example
