## SSD In-Storage Computing for Search Engines

本文作者为UCSD的*J. Wang*以及*Samsung*的*D. Park*，发表于2013年的IEEE Transactions on Computers。本文将web搜索引擎中的查询操作卸载到SmartSSD中，对比了不同操作算法特点得出intersection操作适合卸载到SmartSSD中。difference操作在$|A|\le|B|$的条件下适宜卸载到SmartSSD中，且当intersection ratio更大时优化性能更好。求并操作union涉及大量的内存访问，不适宜卸载到SmartSSD中。SmartSSD可以在除Union以外的操作上减少2-3×的查询延迟和6-10×的能耗。

### Abstract

SmartSSD允许在SSD内部执行应用专用的代码依次利用内部高带宽和高能效的处理器。虽然SmartSSD被应用在诸多领域，何种搜索引擎查询处理操作可以被高效地卸载到SmartSSD中仍是个问题。为此本文选取五个常用操作：intersection、ranked intersection、ranked union、difference、ranked difference。通过将Apache Lucene中以上五个操作卸载到Samsung Smart SSD中并在人造/真实数据集上测试发现可以降低2-3倍的查询延迟以及6-10倍的能耗。

### Introduction

传统的计算架构中SSD只被看作存储单元，主机端执行查询首先从SSD中读取数据然后在主机端的CPU中执行查询任务，这是存算分离的架构。但是近来众多研究表明将数据移向代码无法充分发挥SSD的性能。首先，现代SSD不仅是一个存储设备，其内部还集成了计算单元。在SSD架构中，处理器（通常是ARM系列）被用来承担地址转换、垃圾回收等存储相关的任务，DRAM和Flash芯片被用来存储数据。其次，现代SSD的内部带宽（例如将数据从flash芯片传输至SSD的DRAM中）通常比主机带宽高得多，因此数据访问的瓶颈实际上在于主机带宽。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20221003184619485.png" alt="image-20221003184619485" style="zoom: 50%;" />

SmartSSD接收到主机的查询请求后将从flash中读取必要的数据到内部DRAM中，然后内部处理器执行查询或者查询的部分操作。接着处理的结果（一般比原始数据小得多）将通过带宽较低的主机接口返回到主机端。虽然SmartSSD内部的处理器性能不如主机端处理器，但是其在I/O-intensive以及computationally-simple的应用场景下仍具有如下优势：内部高带宽带来的低I/O延迟、ARM处理器带来的低能耗。本文的贡献在于：

- 从整体系统的角度系统地研究了SmartSSD对搜索引擎的影响；
- 将设计和实现集成到广泛使用的开源搜索引擎—Apache Lucene上；
- 使用合成和真实数据集进行了更全面的实验，以评估性能和能源消耗；

### Background

#### SmartSSD

| <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20221003184702225.png" alt="image-20221003184702225" style="zoom: 67%;" /> | ![image-20221003184724043](https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20221003184724043.png) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

主机接口控制器处理来自主机接口（通常是SAS/SATA或PCIe）的命令，并将它们分发到嵌入式处理器。嵌入式处理器接收这些命令，并将它们传递给闪存控制器。更重要的是，他们运行SSD固件代码用于计算，并执行用于逻辑到物理地址映射的Flash转换层（FTL）。每个处理器都有一个紧密耦合的存储器（例如，SRAM），并且可以通过一个DRAM控制器访问DRAM。闪存控制器控制闪存和DRAM之间的数据传输。由于SRAM甚至比DRAM更快，因此，性能代码或数据被存储在SRAM中，以获得更有效的性能。另一方面，典型或非性能关键DRAM代码或数据。在智能SSD中，由于开发人员可以利用每个内存空间（即SRAM和DRAM），因此性能优化完全取决于开发人员。

图3描述了我们的智能SSD软件体系结构，它由两个主要组件组成：SSD内部的智能SSD固件和主机系统中的主机智能SSD程序。主机智能SSD程序通过应用程序编程接口（api）与智能SSD固件进行通信。智能SSD固件有三个子组件： SSDlet、智能SSD运行时和基本设备固件。SSDlet实现了应用程序逻辑，并响应智能SSD主机程序。智能SSD运行时系统(1)以事件驱动的方式执行SSDlet，(2)将设备智能SSD程序与基础设备固件连接起来，以及(3)实现智能SSDapi库。基础设备固件还可以实现正常的存储设备I/O操作（读取和写）。会话组件管理智能SSD应用程序会话生存期，以便主机智能SSD程序可以通过打开智能SSD设备会话来启动SSDlet。为了支持这种会话管理，Smart SSD提供了两个api，即打开和关闭。直观地说，OPEN启动一个会话并关闭终止一个会话。一旦OPEN启动了一个会话，诸如内存和线程等运行时资源将被分配来运行SSDlet，并将一个唯一的会话ID返回给主机Smart SSD程序。然后，必须将此会话ID关联起来，以与SSDlet进行交互。当关闭终止已建立的会话时，它将释放所有分配的资源，并关闭与会话ID相关联的SSDlet。一旦OPEN建立了一个会话，操作组件将帮助主机智能SSD程序与具有GET和PUTapi的智能SSD设备中的SSDlet交互。此GET操作用于检查SSDlet的状态以及如果结果可用，则从SSDlet接收输出结果。PUT用于在内部将数据写入智能SSD设备，而不需要来自本地文件系统的帮助。

#### Search Engines

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20221003184758339.png" alt="image-20221003184758339" style="zoom: 50%;" />

一个典型的搜索引擎架构如上，包含三个不同的服务器。

- **Front-end web server：**

  前端web服务器与终端用户进行交互，主要是接收用户的查询和返回结果页面。在接收到查询时，根据数据的分区方式（例如， term-based or document

  based分区)，它可能会在将查询转发到索引服务器之前进行一些预处理。

- **Index server：**

  索引服务器存储倒置的索引，以帮助有效地回答查询。反向索引是任何搜索引擎中的一种基本数据结构。它可以有效地返回包含查询术语的文档。它由两部分组成：*dictionary*和*posting file*。

  - **dictionary : <term,docFreq,addr>**。term代表关键词，docFreq代表包含该关键词的文档的数量，addr指向posting file中的倒排列表；
  - **posting file : <docID,termFreq,pos>**。该文件中的每一项称为倒排列表。docID为文章的标识符，termFreq为关键词在文章中出现的次数，pos记录了关键词在文章中出现的位置。

  索引服务器接收到查询后返回top-k个相关的文档，具体经过上图X1-X5的操作。

- **Document server：**

  文档服务器存储实际的文档（或网页）。它从索引服务器接收查询和一组文档id，然后生成特定于查询的文档片段，处理流程如上图S1-S5。

**我们主要关注索引服务器，并将文档服务器留给以后的工作，根据经验，基于SSD的搜索引擎中索引服务器的查询处理将比文档服务器花费更多的时间。**此外，我们选择Apache Lucene作为我们的实验系统，因为Lucene是一个著名的开源搜索引擎，在工业中被广泛采用

### SmartSSD for Search Engine

#### Design Space

为了理解何种查询操作可以更好地在SmartSSD中执行，我们必须了解SmartSSD的优势和劣势；

- **Opportunities：**在SmartSSD内部执行I/O操作很快，因为SSD内部带宽数倍于主机I/O接口带宽，且I/O延迟与主机端相比要低，因为从SSD到主机端的I/O经过操作系统会涉及中断操作、上下文切换、以及文件系统。
- **Limitations：**低频处理器性能低、一般SSDs控制器没有充足的缓存所以访问设备中的DRAM要比主机端慢的多。

因此在SSDs内部不适宜执行CPU密集型以及Memory密集型的应用。如果应用的性能瓶颈在I/O那么SmartSSDs可以缓解其性能瓶颈。例如下图中在常规SSD中执行Lucene系统查询，发现大部分的时间都花费在了I/O上，因此将查询卸载到SmartSSDs中可以减小I/O时间（**CPU时间略有增加 还是一个卸载的trade-off问题 在I/O带宽和CPU性能之间trade-off，可否实现automation？**）。基于此，文章分析了Index server中的查询步骤来探究哪些步骤在SmartSSD中执行可以同时降低查询延迟和能耗。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20221003184859123.png" alt="image-20221003184859123" style="zoom:80%;" />

- **X1 Parse query：**解析查询涉及到许多cpu密集型的步骤，如标记化、符号化， 词形还原，因此不适合卸载此步骤；
- **X2 Get metadata：**元数据本质上是一个键-值对。该键是一个查询术语，其值是关于该术语的磁盘上倒排表的基本信息，通常包括：列表存储在磁盘上的偏移量、列表的长度（以字节为单位）、以及列表中的条目数量。字典文件存储元数据。我们为字典文件构建了一个类似于btree的数据结构。由于获取元数据只需要很少（通常需要1∼2个）I/O操作，所以我们不卸载此步骤。
- **X3 Get inverted list：**获取倒排表。每个倒置的列表都包含一个包含相同术语的文档列表。在接收到查询后，Smart SSD将磁盘中的反向列表读入主机内存4，这是I/O密集型的。如图5所示，如果Lucene在常规ssd上运行，步骤S3需要87%的时间。因此，我们希望将此步骤卸载到SmartSSD中。
- **X4 Execute list operations：**将倒排表加载到host内存中是为了高效地执行例如相交这样的计算操作，因此X3和X4都应该被卸载到SmartSSD中。这同时需要决定哪种操作事宜被卸载到SmartSSD中。我们设置了一个简单的准则，即操作输出的数据量应当小于输入的数据量，否则SmartSSD无法减少数据迁移。
  - 假设$A$和$B$是两个倒排表且$A$更短。
    - **Intersecton：**$|A\cap B|\ll|A|+|B|$，在Bing search中有大约76%的查询，求交后数据量比原数据量比最短的倒排表小两个数量级。
    - **Union：** 求并操作和两个倒排表的数据量差不多。$|A\cup B|=|A|+|B|-|A\cap B|$。大部分情况下将Union操作卸载到SmartSSD中并不高效。
    - **Difference：** 在$|A|<|B|$的条件下，对于$|A-B|=|A|-|A\cap B|<|A|\ll|A|+|B|$，因此将$A-B$在SmartSSD中执行可以减少迁移的数据量。相反，$|B-A|=|B|-|A\cap B|\approx|B|\approx|A|+|B|$，因此求差操作可以作为查询卸载的候选操作。
- **X5 Compute similarity and rank：**利用排序算法计算top-k个结果是CPU密集型操作，但是当计算结果很大时考虑到只需要返回较少的top-k个结果，即如果将这一操作放到SmartSSD中可以节省I/O时间。**因此对于X5我们考虑卸载和不卸载两种选择**。

综上作者考虑将以下五种操作卸载到SmartSSD。对于不涉及排序的操作，只有X3和X4在SmartSSD中执行，对于涉及排序的操作X3，X4，X5均在SmartSSD中执行，其他情况下X1和X2都在主机端执行。

![image-20220927150941515](https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927150941515.png)

#### System Co-Design Architecture

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220928105545262.png" alt="image-20220928105545262" style="zoom: 33%;" />

查询的整个处理流程首先是Search engine负责接收查询。一旦接收到查询$q(t_1,t_2,t_3,...t_u)$,步骤S1将查询解析为u个关键词并在S2步骤获得u个关键词的元数据。然后通过OPEN API将元数据信息发送给SmartSSD。SmartSSD然后利用得到的元数据信息将u个倒排表加载到其内部的DRAM中，DRAM通常有几百MB。当所有倒排表都被装入DRAM中，SmartSSD开始执行求交操作（**可否流水线化？**），一旦操作完成计算结果将被放入输出缓冲池中并准备返回到主机端。主机端通过GET API以心跳的方式监测SmartSSD的状态，我们将轮询间隔设置为1ms。一旦主机端的搜索引擎接收到了求交运算的结果将会执行步骤S5来计算top-k个结果返回给用户，在某些情况下S5也会在SmartSSD中执行。

### Implementation

这一部分描述了将查询卸载到SmartSSD中的具体细节。

#### Intersection

假设共有u个倒排表$L_1、L_2、...L_u$，我们首先需要将u个倒排表加载到设备内存中，然后执行in-memory的列表求交算法。有很多存内列表求交算法例如基于排序-合并、基于跳表、基于哈希、基于分治、基于适应性算法、基于分组。我们选择基于适应性的列表求交算法，主要考虑到适应性算法在理论和实践上表现得都很好并且Lucene内部也是使用了适应性算法来进行求交运算。算法首先设置一个游标指向$L_u$的第一个元素，不断轮询所有的倒排表，利用二分寻找大于等于游标的最小元素，如果找到的结果与游标相等，则增加计数器，如果计数器到达$u$则表明游标所代表的元素在所有列表中都出现，则把该元素加入最后的结果集。如果在当前列表中搜索得到的元素与当前游标所指元素不等，则将游标指向该元素。

算法1的性能依赖于第6步，即如何确定当前游标代表的元素是否在搜索的列表中。在实现中我们主要使用了二分搜索，**然而在两个列表大小相似时，线性搜索会比二分搜索更快**。在本文实现中，当两个列表的大小比小于4时(这个值基于作者经验设置)使用线性探测，实际中适应性算法将切换成排序-合并算法。为了公平对比，在使用常规SSD时我们也对Lucene进行了修改，使其可以在适应性算法和排序-合并算法之间进行切换。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220928144115727.png" alt="image-20220928144115727" style="zoom: 33%;" />

算法1要求设备内存可以存储所有的lists，对于list太大存放不下的情况我们有两种解决方式：

- **Solution1：**

  假设设备内存容量M比最短列表的容量的两倍还要大，它将最短的列表L1从闪存加载到设备内存中。对于L2中的每个页面，它检查该项是否出现在L1中，并将结果存储在中间结果缓冲区中。对于L2中的每个页，检查其是否在L1中并将结果存放在中间结果buffer中。因为求交以及求差的结果大小比$|L1|$小，所以两倍于L1大小的设备内存足够了。随后，L1和L2相交的结果再和L3求交。这个过程一直处理完所有的列表。

- **Solution2：**

  不做任何假设，即对设备内存大小无要求。主要思路为将最短的列表L1均匀地划分成m块$L_1^1,L_2^2,L_3^3,...L_m^m$，这些块的大小可以存放入设备内存中。剩下的u-1个列表依次和$L_1^1,L_2^2,L_3^3,...L_m^m$求交。对于每一个$L_1^i,L_2^2,L_3^3,...L_m^m$都按照solution1的方法执行，最后将结果合并并返回给用户。

#### Union

采用基于排序-合并的算法执行求并操作。首先对于每一个列表都设置一个下表指针$p_i$，初始化为0。每一步扫描所有的$L_k[p_i]\ for k=1...u$，然后找出最小值$minID$,将其加入结果集然后将所有等于最小值的$p_i$向前移动一位。一共需要$2u$次内存访问，第一遍寻找最小值，第二遍扫描旨在迁移指针。结果中$|L_1\cup L_2\cup L_3\cup...\cup L_u|$个元素最终大约访问$2u|L_1\cup L_2\cup L_3\cup...\cup L_u|$次内存。后续将讨论Union操作对系统性能的影响。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220928144014350.png" alt="image-20220928144014350" style="zoom: 33%;" />

#### Difference

求差运算比较简单，对于$A-B$，我们检查$A$中的元素是否在$B$中，如果在则提出，不在则加入结果集。对于探测使用二分查找，对于列表大小相差四倍以上的，我们采用线性探测。

#### Ranking

采用常见的传统计算方式BM25，其实就是$tf-idf$模型。相关符号意义如下：

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220928170538311.png" alt="image-20220928170538311" style="zoom: 80%;" />

公式（1）本质上就是累加查询中每一个关键词与文档的相似度乘上关键词在查询中的频率。公式（2）计算了一个关键词与文档的相关度。公式（2）中每一项都可以提前计算，考虑到SmartSSDs中的处理速度有限，我们存储了$Similarity(t, d)$。我们在SRAM中的一个heap-like数据结构中维护top个计算结果。SRAM大概有32KB，堆结构保留top-10个结果，大小为100Bytes。
$$
\operatorname{Similarity}(q, d)=\sum_{t \in q}( Similarity (t, d) \times q t f)
$$

$$
\operatorname{Similarity}(t, d)=t f \cdot\left(1+\ln \frac{N}{d f+1}\right)^2 \cdot\left(\frac{1}{d l}\right)^2
$$

### Analysis

本节简要分析了在性能和能量方面对卸载SmartSSD中的搜索引擎操作的权衡。（**之前一篇disk论文中也有建模**）

- **Performance analysis：**

  假设$T、T^，$分别表示操作在SmartSSD和host（regular SSD）中执行的时间。$T$可被分为三个部分：将列表从flash chip读取到设备内存的时间$T_{I/O}$，操作执行时间$T_{CPU}$,以及将计算结果传回主机端的时间$T_{send}$。主机端操作类似。我们可以得到：$T=T_{I/O}+T_{CPU}+T_{send}\ 、T^,=T^,_{I/O}+T^,_{CPU}$。根据前边对SmartSSD的特性介绍我们有$T_{I/O}<T^,_{I/O}$，但是$T_{CPU}>T^,_{CPU}$。由于求教运算结果很小，$T_{send}$可以忽略不计。

- **Energy analysis：**

  假设$E、E^,$分别是采用SmartSSD和主机端运行操作的能量消耗(J)，$P、P^,$分别是SmartSSD的CPU和主机端CPU的功率(W)，有$P$一般小于$P^,$的3-4倍。考虑$E=T\times P$，所以大概率SmartSSD的能耗更低。

### Experimental Setup

#### 数据集

真实数据集来自搜狗的web数据以及query日志。人造数据相关说明如下，列表被划分成两类，短列表和长列表且每一类列表长度相同。List Size指的长列表B的大小。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151232247.png" alt="image-20220927151232247" style="zoom: 50%;" />

#### 软硬件环境

主机是一个商品服务器与英特尔i7处理器（3.40 GHz）和8 GB内存运行Windows 7。该智能SSD有一个6Gb的SAS接口和一个400 MHz的ARM Cortex-R4四核处理器。该设备通过一个6Gb的SAS主机胸部适配器（HBA）连接到主机。主机接口带宽（SAS）约为550 MB/s。内部带宽约为2.2 GB/s。与普通SSD相比，智能SSD具有相同的硬件规格和基本固件。C++ 0.9.21version。我们通过WattsUp来测量功耗：当系统足够稳定（即空闲）功率和运行时功率分别设为W1和W2（以瓦茨为单位），t为查询延迟。用（W2−W1）×t计算能量。

### Experimental Results

#### Intersection

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151256591.png" alt="image-20220927151256591" style="zoom: 67%;" />

可以看到SmartSSD极大的减小了I/O时间，但是受限于其内部低频率CPU，CPU时间有所增加，最终查询延迟减少了2x。接着我们将S3（读取倒排表）和S4（求交运算）两步操作拆解。将数据从flash chip传输至DRAM、内存访问、计算占据了S3-S4中的大部分时间。可以通过提高内部带宽、使用DMA或更多缓存、提升处理器性能缓解。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151321276.png" alt="image-20220927151321276" style="zoom:67%;" />

以下为不同列表长度以及不同列表大小比率的实验结果，可以看到SmartSSD与传统SSD相比在查询延迟以及性能上都有优势。

| <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151425838.png" alt="image-20220927151425838" style="zoom: 50%;" /> | <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151439647.png" alt="image-20220927151439647" style="zoom: 50%;" /> |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

图11为不同求交比率的实验结果，可以看到虽然SmartSSD优于传统SSD，但是随着求交结果的增大查询延迟和能耗并没有变高。因为列表A默认0.1MB，列表B为10MB，作者将列表A、B的大小都设置为10MB增加结果的数据量，实验结果如图12。

| <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151501078.png" alt="image-20220927151501078" style="zoom: 50%;" /> | <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151518522.png" alt="image-20220927151518522" style="zoom:50%;" /> |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

图13展示了改变列表数量（可以理解为查询中关键词集合的数量，因为一个关键词对应一个倒排表）下的实验结果。随着操作涉及的数据量增加，查询延迟和能耗均不断增加。当列表数量从2{A，B}变成3{A，B，A}时查询延迟变化不大，因为A的大小是B的1/100。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151543747.png" alt="image-20220927151543747" style="zoom: 33%;" />

求交操作可以卸载到SmartSSD中，特别是当列表数量多，列表长，列表大小倾斜，交叉比低时。

#### Ranked Intersection

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151606410.png" alt="image-20220927151606410" style="zoom: 33%;" />

S3、S4、S5均被卸载到SmartSSD中。与上边的Intersection相比，将S5也卸载到SmartSSD中可以减少需要向主机转移的数据量，但是会增加SmartSSD中的计算成本。到但时当结果非常小时差别不大。图14为将A和B列表设置相同大小考虑不同结果比率得到的结果。我们发现虽然有所增长但是增长的不明显例如当ratio从10%增长到100%，图14中SmartSSD查询延迟增加了65ms，但是图12中增加了163ms，这与额外的数据移动相关。对于排序求交运算，由于只返回排名最高的结果，因此需要移动的数据量更少。

#### Difference

当较短的列表A与较长的列表B求差时，相对于B-A，SmartSSD更加受益与A-B。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151711456.png" alt="image-20220927151711456" style="zoom: 50%;" />

上图为在真实数据集上的实验结果。图16为不同$f=\frac{|B|}{|A|}$下实验的结果。我们将$f$从0.01变化到100。$f<1$意味着A比B更长，反之A比B更短。实验结果可以看出当$f=0.01、0.1$时SmartSSD具有更长的查询延迟。例如$f=0.01,\ \frac{|A-B|}{|A|+|B|}=98.6\% $，因此当$|A|>|B|$时不适合将A-B卸载到SmartSSD上，因为不能很好的降低数据迁移。当$f\ge 1,|A|\le|B|,|A-B|\le|A|\le\frac{|A|+|B|}{2}$，也就是说将A-B卸载到SmartSSD中最少可以减少50%的数据迁移。图17反映了不同求交比率对作差运算的影响，考虑到ratio越大，A-B的结果越小，结果也是合理的。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151743650.png" alt="image-20220927151743650" style="zoom: 50%;" />

#### Ranked Difference

以下为将S5也卸载到SmartSSD中时排序做差操作的实验结果。

| ![image-20220927151809078](https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151809078.png) | ![image-20220927151822829](https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151822829.png) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |

#### Ranked Union

表5显示了使用真实数据的实验结果：智能SSD在查询延迟方面比常规SSD慢约1.7×。这是由于内存访问过多造成的，但是SmartSSD在功耗方面仍然具有优势。由于内存访问过多，将排名求并操作卸载到SmartSSD上是不划算的。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151835422.png" alt="image-20220927151835422" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/In-Storage%20Computing%20for%20Search%20Engines/image-20220927151857158.png" alt="image-20220927151857158" style="zoom:50%;" />

### Conclusion

本文本质上还是针对传统文本搜索模型tf-idf进行了优化，对于其他搜索模型没有讨论。当然距离这篇paper发表已经过去6年了，在这期间SmartSSD也在发展，后期可以调研可以加速的更多应用。
