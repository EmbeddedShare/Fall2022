## FPGA-based Compaction Engine for Accelerating LSM-tree Key-Value Stores （ICDE）

本篇论文来自香港中文大学Xuan Sun，发表于ICDE 2020。 作者发现在大量写的工作负载下，LSM-tree中频繁的compaction操作会占用大量CPU资源，导致kv store数据持久化被阻塞，影响系统性能。针对这一问题，作者使用FPGA作为协同加速器将compaction操作从CPU中部分分离出来，并针对FPGA特点提出了key-value分离、index-data分离策略以优化FPGA的流水线工作效率。

### 1. 背景

1. 如图1所示，主线程负责处理用户的写读请求，compaction线程负责调度和请求compaction任务。在大量写的工作负载下，LSM-tree中频繁的compaction操作可能导致使得用户数据持久化被阻塞的问题。本文使用FPGA对基于LSM-tree的KV数据库进行compaction操作的加速。使用FGPA作为加速器主要面对如下挑战：1. FPGA上compacation单元的设计使其拥有比CPU更高的性能；2. 针对FPGA特性的流水线设计以充分发挥性能。3.为了与数据库的无缝衔接，软硬件层次之间的接口应被合理设计以防止数据损失；4. 主机端需探究**如何以及何时**使用FPGA加速模块来提高整体性能。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210041612014.png" alt="image-20220926211908754" style="zoom:80%;" />

2. LSM-tree的compaction操作分2类，第1类是将内存中的MemTable转化SSTable，第2类是当Level i的SSTable数达到一定阈值，通过sort-merge操作在level i+1产生新的SSTable。除了level 0外，其余level的SSTable都是有序的。每个SSTable都有1个kv pair组成的index block，其中key记录相邻data block的中间位置，value记录data block的**大小和偏移量**。当完成一个data block的merge后，需要先到index block获得下一个data块的元数据，再完成merge操作。

### 2. 系统设计

1. 作者实验的整体系统架构如Figure1所示。主机端中主线程负责处理用户的读写请求，compaction线程调度compaction任务并通过PCIe总线下发到FPGA板上。FPGA板由FPGA芯片和DRAM组成。

2. 当Level i SSTable达到阈值后触发merge操作，background threads执行如下操作：1. CPU收集Level i 和 Level i+1中需要被compact的SSTable元数据；2.background线程计算输入的数量，对于Level 0，由于其SSTable中的key存在重叠，因此其输入数量等于Level 0中SSTable的数量。对于其他Level，由于其SSTable都是有序的，因此其参与的多个SSTable可以被连接为1个大的SSTable，其输入数量为1；3. CPU将输入的SSTable读入到1个了连续的内存块中，并为即将生成的新SSTable在内存中分配空间；4. 这些compaction操作的输入数据通过**PCIe总线，以DMA**的方式被传输到FPGA板上的DRAM；5. 输入数据读入到DRAM后硬件compaction引擎开始工作，从DRAM中读取数据；6. 当达到FPGA芯片的存储限制后，compaction操作的部分结果会先被写入DRAM中；7.当compaction操作完成后CPU接受end信号并将新生成的SSTable读入到内存；8. CPU将新生成的SSTable写入硬盘，并完成compaction后的处理工作如记录key的范围。

    ![image-20220926211908754](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210041612014.png)


### 3. 硬件实现

这部分首先介绍FPGA compaction 引擎基本的流水线设计，接着根据FPGA特点提出1. Decoder/Encoder 2. Key-value separation 3. Data Transmission 3种优化方式。

##### 基础流水线模型

FPGA compaction引擎如Figure2所示，由Decoder、Comparer、Encoder3部分组成。**Decoder**的伪代码如Algorithm 1所示，其3层嵌套循环分别遍历SSTable、index快中的kv pair(需要index中的大小和偏移量来确定下一个data block的位置)、data块中kv pair(解压缩)。解压后的kv放在key-value buffer中。**Comparer**模块包含key比较模块和验证模块，key比较模块选出最小的key值；由于SSTable的中的每条数据带有标志位，验证模块通过检查标志位判断该kv pair是否有效，comparer结束后会将drop标志传送给key-value transfer，kv transfer根据drop标志判断将该条kv数据丢弃或者传送给Encoder。此外，comparer还需将选出的最小key所在的输入的输入编号Input No传送给KV transfer***(有啥作用？)***。**Decoder**将key进行压缩。当Output Buffer中的数据块大小达到一定阈值后被写入DRAM，并将该data block的元顺据记录到index block，index block由BRAM(FPGA on-chip RAM)实现；当DRAM中data block的累计大小达到一定阈值表示1个SSTable形成，BRAM中的index block被写入到DRAM中，新的SSTable返回给主机端。

<img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209281549061.png" alt="image-20220928154921023" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209281552909.png" alt="image-20220928155233871" style="zoom:80%;" />

##### Decoder/Encoder优化

根据FPGA特点针对Decoder/Encoder的优化如Figure 3所示。Decoder划分成index block decoder和data block decoder，原因如下：1. 基本Decoder中读指针需要在index block和data block中转换(1个data block解压缩完成后需要到index block中获得下一个data block的元数据)，将二者的decode分开可以使得decode以流水线的方式进行；2. 考虑到从FPGA内存(1个周期)和DRAM(7-8个周期)中读取数据所需的时间差，优化后的方案可以在index block decoder解析出data block的元数据后将整个data block按FIFO的方式写入FPGA(降低到decode 1个kv pair 所需要的读请求次数的时间周期***（第2点不是很理解）***。

类似的，Encoder分成Index Block Encoder和Data block Encoder。在基本模型中，index block缓存在BRAM中直到所有data block encode完成，该方式增加SSTable传输时间和BRAM的使用率。而优化方案可以在新的data block生成后就立即将相应的index block写入到DRAM，规避传统模型的缺点。

<img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209281642152.png" alt="image-20220928164223099" style="zoom: 67%;" />

##### key-value 分离优化

在传统的LSM-base kv store中，kv键值对被当作一个整体处理，但实际上在compare阶段不需要用到value，较长的value值会增加执行时间，因此作者提出key-value分离的优化的策略，如Figure4所示。相关的优化工作如下：1. decode后的kv pair被分成3个数据流(key、key copy、 value)，key送到compare模块中去获得最小的key，key copy和value送入到Transfer module中，等待根据compare模块的结果被选择；2.根据drop flag标志同时判断是否drop掉key和value；3.value transfer的结果直接送到输出缓存，key transfer的结果送到Encoder中编码。

<img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209282033008.png" alt="image-20220928203339954" style="zoom:80%;" />

####  数据传输优化

1. value传输位宽

   数据优化前，各个阶段的大概周期如表2(忽略index block的时间)。其中L表示长度，N表示N路输入。comparer中，$\log _2 N$表示N组输入选取最小值所需的次数，2表示1次验证＋1次key读取。使用周期最长的阶段决定了系统的平均执行时间，而value往往比key大得多，因此瓶颈在于value数据流的传输。设置数据位宽为V，则value传输时间近似于$\frac{L_{\text {value }}}{V}$。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209282056061.png" alt="image-20220928205659014" style="zoom:80%;" />

   数据优化后各阶段的执行时间如表3所示。此时需要最长时间周期的阶段为decoder或comparer。当$L_{\text {key }}+\frac{L_{\text {value }}}{V}>\left(2+\left\lceil\log _2 N\right\rceil\right) \times L_{\text {key }}$时，即$L_{\mathrm{key}}<\frac{L_{\text {value }}}{\left(1+\left[\log _2 N\right]\right) \times V}$时decoder为执行瓶颈，否则是comparer。

   ![image-20220928211826276](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209282119052.png)

2. FPGA芯片输出输入位宽

   FPGA通过AXI协议从DRAM中读取数据和向DRAM写入数据，AXI允许最大64byte/cycle。设FPGA通过AXI读速度为$W_{\text {in }}$，decoder消耗速度为V，V<=$W_{\text {in }}$。因此，每个Decoder前都插入一个Stream Downsize来缩小输入流；相同的，每个output buffer后增加1个Stream Upsizer。





### 4. 软件与硬件compaction引擎的集成

1. 由于FPGA资源限制，FPGA最多只允许N路输入，因此当Level 0中的SSTable个数大于N-1，compaction任务由软件SW compaction模块完成。Figure 6展示compaction任务的工作流程。当执行软件compaction时需检查内存中的Immutable MemTable是否需要被flush到硬盘，如需要flush操作则compaction需先暂停等待flush操作完成。而当使用FPGA进行compaction操作时，当compaction任务被下发到FPGA，主机端的flush操作可以并行完成。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210041613492.png" alt="image-20220930085644723" style="zoom:67%;" />

2. FPGA与内存的输入输出接口的数据应该被统一，考虑到index块和data块在Encoder和Decoder中被分开处理，因此data块和index块应被存放在内存的不同位置，如figure 7所示。Index块可以连续存放，由于data block以Win-byte/cycle的速度读取，因此读入的data block需按Win-byte的偏移量对齐，相同的输出的data block按Wout-byte的偏移量对齐。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209300938925.png" alt="image-20220930093823829" style="zoom:80%;" />

3. 此外，FPGA compaction引擎还需要额外的信息，如Figure 8所示。MetaIn Memory存放SSTable的数量和每个SSTable 的index block、第一个data block的偏移量。MetaOut Memory中维护每一个SSTable中最大和最小的key值，以及该SSTable的大小。

![image-20220930094721849](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209300947887.png)



### 5. 实验评估

作者用LevelDB和LevelDB-FCAE(FPGAbased Compaction Acceleration Engine)来进行试验。FPGA compaction引擎使用 带有16GB DRAM的Xilinx Kintex UltraScale FPGA KCU1500 card 实现，与主机使用PCIe gen3*16通信。Level运行在2核CPU上而LevelDB-FCAR运行在1核CPU+FGPA板上。使用db_bench和YCSB benchmark。

##### 评估2-input FCAE

1. 我们研究V的变化对**compaction加速**的影响。Table V展示不同Value长度和V下compaction速度，其中速度按 Size of input SSTables / Kernel compaction time计算。Figure 9展示不同参数下compaction速度的加速倍率。实验结果表明CPU和FPGA的compaction速度都随着value长度的增大，然而FACR的增加幅度更大，这得益于key-value分离和数据传输带宽的增加；V(FPGA上的数据传输位宽)的增加也使得compaction加速比率的增加。特殊情况是V=0和V=16，value长度为1024时，加速比率略有下降。这是由于上文所讲的当$L_{\mathrm{key}}<\frac{L_{\text {value }}}{\left(1+\left[\log _2 N\right]\right) \times V}$时瓶颈在与Data block Decoder,否则在comparer。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301025733.png" alt="image-20220930102524697" style="zoom:80%;" />

   

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301027838.png" alt="image-20220930102704799" style="zoom:80%;" />

2. 使用db_bench评估写**吞吐量**。Figure 10展示不同数据大小下的写吞吐量，可以看出LevelDB写吞吐量下降的速度比LevelDB-FCAE更加剧烈，这是因为当数据量增加时资源竞争导致compaction线程不得不暂缓将immutable刷入磁盘，转而更加频繁的处理merge操作，而对于FPGA-FCAE，FPGA compaction引擎缓解了CPU的compaction任务。Table VI和Figure 11分别展示了不同value长度和V下的写吞吐量、和写吞吐量的加速比率。对于原始LevelDB，value长度的使得写吞吐量性能略有下降，这是因为即使CPU具有高频率但数据的移动依旧影响compaction表现。当value长度固定，不同的V(FPGA中数据传输带宽)也对写吞吐量有影响，当value值较小时，FCAE的加速比率没有太大的变化，因为此时瓶颈在comparer(具有比较稳定的周期3*Lkey

   )；当value长度较长时，瓶颈在于Decoder(Lkey + Lvalue/V)，随着V的增大，时间减小，加速倍率增大。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301110844.png" alt="image-20220930111039802" style="zoom:80%;" />

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301124042.png" alt="image-20220930112445996" style="zoom:80%;" />

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301125486.png" alt="image-20220930112509447" style="zoom:80%;" />

 ##### 评估多路输入-input FCAE

Level 0的compaction包含最多的SSTable情况 ，8个Level0的SSTable和1个Level 1的SSTable，因此N最大为9，对于其他lazy compaction(非level 0但是其具有重复的key值)，当输入数量不大于9时使用FPGA进行compaction，否则使用CPU进行compaction。

1. FPGA配置。当N=9时，若Win和Wout设置为最大值64，则FPGA资源将被耗尽。考虑到输出只有一路而输入由多路，作者设置Wout为64，降低Win和V。Table VII列出不同配置下FPGA资源使用情况，采用9-input FCAR设置Win=8，V=8。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301448217.png" alt="image-20220930144859170" style="zoom:67%;" />

   Figure 12展示不同value长度下compaction的吞吐量。可以看出9-input FCAE在速度上有所衰减相对于2路FCAE，但差距随着value长的增加和缩小。因为瓶颈由comparer转移到decoder。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301517358.png" alt="image-20220930151703321" style="zoom:80%;" />

   Figure 13显示 FCAR相对于CPU基准的加速比率随着value值增大。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301519996.png" style="zoom:80%;" />

   2. 不同数据大小的影响。Figure 14显示由于compaction代价的增加，数据大小增大使得LevelDB和LevelDB-FCAE的写性能显著降低。随着数据的增大levelDB-FCAE的加速也在下降，最后维持在2.5倍左右。

      <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301630145.png" alt="image-20220930163021092" style="zoom: 50%;" />

   3. 评估LevelDB设置(key长度、value长度、数据块大小、等)的影响。Figure15 a 显示加速比率随着key长度线性降低；类似的，加速比随着value长度增加而增加；Figure15 c显示LevelDB和LevelDB-FCAE和块大小无关；leveling ratio比率定义为**Size(Level i +1) / Size(Level i)**，Figure15 d显示随着leveling ratio比率的增大加速率下降，这是因为leveling ratio越大，compation触发得越不频繁，因此FPGA的作用就不明显。

      <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301637229.png" alt="image-20220930163752174" style="zoom:80%;" />

##### 在YCSB上评估测试

使用YCSB进行测试，对每个负载的操作次数为20M。如Figure 16所示，LevelDB-FCAE在各类负载下都比LevelDB表现更佳，即使在只读负载下吞吐量也并没有衰减，因为并没有对原始的存储结构进行改变。Figure16显示写占比越高加速比率越高，对只读负载达到2.2倍。

<img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209301938830.png" alt="image-20220930193851773" style="zoom:67%;" />



### 结论

这篇文章提出一种使用FPGA作为compaction引擎加速基于LSM-tree的KV store的方案，将compaction下发给FPGA上的compaction引擎执行。同时优化index块和data块的流水线模型，并提出key和value分离处理的方案。作者使用统一的内存接口任务调度策略将FPGA compaction引擎与LevelDB相结合。实验结果表明相比基于CPU的compaction速度可以提高92倍，写吞吐量最高提升6.4倍。