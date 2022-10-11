## REMIX: Efficient Range Query for LSM-trees

本篇文章由伊利诺伊大学芝加哥分校和德克萨斯大学阿灵顿分校合作完成，作者为wenshaozhong。作者发现采用tiered compaction策略的LSM-tree中同一Level下 的sorted run具有重叠部分，范围查询往往要查询和比较多个table file，极大影响读性能。针对此问题作者提出一个空间上高效的索引数据结构REMIX，通过REMIX采用稀疏索引和二分查询的方式快速定位到目标key的位置，并在无需key值比较的基础上连续读取相邻的key，完成范围查询。使用REMIX和tiered compaction策略可以使LSM-tree在保持较低写放大的同时提高范围查询的性能。

### 1. 背景

LSM tree中范围查询可能需要不断访问不同的table，导致大量的IO和key的比较计算开销，影响系统性能。LSM tree维护sorted run需要大量重写已有数据，而如今的存储技术已经大大提高随机读取的效率，因此作者发现LSM tree中kv pair并不一定需要在物理上有序，而是可以逻辑上有序，来避免大量的重写工作。在此基础上作者提出REMIX(Range-queryEfficient Multi-table IndeX)，使用一个空间高效的数据结构来记录全局的排序视图，以此在发挥LSM tree高效写优势前提下提高LSM tree范围查询的效率。

LSM中常见的compaction策略有2种。如Figure 1所示，leveled compaction在每一层维护1个sorted run，每一个sorted run包含一个或多个table，该策略将相邻两层的中重叠的table进行sorted-merge，并将新产生的table放在较大的一层，也由于其对重跌table数量的严格控制，leveled策略有较好的读性能，但写放大问题较严重；如Figure2所示，tiered compaction允许多个重叠的run存放在1个level，只有当某一层中sorted run的数量超过某个阈值，将该层中的sorted run合并成一个新sorted run放在下一层，并不会重写已经存在的数据，也因此其查询开销较大，写入速度较快。

![image-20221005194158482](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051941530.png)

传统LSM tree中范围查询由迭代器实现。如Figure 1、2右上角所示，首先通过二分法找到每一个run中大于seek key的最小key，并用游标标记这些key，接着比较这些游标所指向的key的大小，选出最小的key后向下移动该游标，进一步重复比较游标上key的大小重复此上述操作；

### 2. REMIX

在范围查询时，实际上数据库建立了一个**有序视图**，这个有序视图具有不变性，直到有table file被替代或删除，然而现有的LSM-tree kv store都没有利用这个有序试图的**继承不变性**，而是在每一次范围查询时重新建立查询视图并在查询完成后随即抛弃。REMIX的动机在于利用table file的不变性，维护排序视图并重复使用该视图提高读性能。

1. Figure 3展示1个包含3个sorted run的有序视图，其中蓝色箭头指向形成有序的排序序列。构建REMIX首先将有序视图分割为多个segment，每个segment包含固定数量的key，每1个segment包含1个anchor key，多个cursor offset(游标偏移量)、多个run selector(run 选择器)，其中anchor key表示1个segment中最小的key(如下图红色数字)，所有segment 的anchor共同组成该有序视图的**稀疏索引**；每个cursor offset对应1个segment和1个run，记录该run中大于等于所在segment的anchor key的最小的key；run selector序列组成了顺序访问sort view时的run访问顺序。

    REMIX迭代器包含2部分：多个游标和1个当前指针。迭代过程如下：1. 在anchor key上使用二分查询法找到包含目标key的segment，满足anchor_key ≤ seek_key；2. 游标定位到cursor offset所指向的位置，current point 指向该segment 的第1个run selector；3. 遍历操作。run selector和cursor offset共同确定了1个key，再获取该目标key后，该目标key所在run的游标向右移动，同时current point指向下一个run selector，重复该过程。

![image-20221006143927615](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061439575.png)

2. **segment内部二分。**如上所述，通过二分法找到目标segment后，在segment内通过顺序遍历来找到目标key。segment增大以减小anchor key的数量，使二分查找更快，但会减慢segment内部的查询操作，segment内部key也是有序的，因此作者提出在segment内使用二分查询目标key。二分查询首先要解决segment中key随机访问的问题，因此作者引入了occurrence，表示在segment中，和某个key相同的run selector在当前位置之前出现的次数(对于某个key，在其所在run中，出现在该key之前的key的个数)。通过run selector和occurrences可以随即访问1个key(从该run的第1个key开始跳过occurrence个key(并不用去读取这个key))。occurrences可以通过SIMD指令被CPU快速计算出来。因此通过二分run selector和occurrences数组的下标可以获得下一个目标key的位置，再根据run selector和occurrence读取该key。在segment内部使用二分法查询下文被称为满二分法，否则成为部分二分。

   **IO优化。**segment内二分查询路径上的key可能位于不同的data block中，导致需要多次IO。因此当找到二分查询路径上的某个key，可以利用该key所在数据块上的key进一步缩小二分查询。例如查询71，第一次二分查询比较R3的43，接着比较R3的其他key，通过83将二分的范围缩小为[43，83]，减少IO读取数据块的次数。

   ![image-20221006152003532](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061520583.png)

3. REMIX对范围查询的性能提升主要有3方面。1. 传统LSM-tree的范围查询需要在每个run中都进行1次二分查找，而REMIX可以在多个run中只进行1次二分查找；2. REMIX通过run selector和每一个run上cursor的移动直接获取下一个key，而传统LSM需要读取多个overlap的sorted run的key进行比较来获得下一个key；3. 对于目标segment中的二分查询，REMIX不会访问不在搜索路径上的run。最好情况下，如果范围查询的key都位于同一个run中，REMIX也只需要读取1个run。而传统LSM-tree需要访问所有的run(访问多个data block)。

4. **分析存储代价**。REMIX元数据包含anchor key、cursor offset、run selector。设每个segment中最多存放数量D的key，L为anchor key的平均大小，则每个key需要L/D的大小存放anchor key；cursor offset大小为S字节，H为run数量，则每个key需要SH/D字节大小存放cursor offset；1个run selector需要log2H比特的空间，因此REMIX的空间代价为：

   ![image-20221006161801471](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061618513.png)

   表1展示了不同负载下REMIX空间代价，以及LevelDB、RocksDB中block index和布隆过滤器的空间代价作为对比。

   ![image-20221006161904342](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061619387.png)

### 3. RemixDB

RemixDB采用具有高效写特性的tiered compaction策略，分区存储有利于空间局部性的负载，因此Remix将key空间划分为没有key overlap的partition，每一个partition由1个REMIX索引。***(分区是如何划分的？当immutable memtable中有位于两个分区的key range时怎么compaction)***  Figure 5展示RemixDB的组成，compaction操作会生成包含新旧table file和新REMIX的新版本partition，旧版本随后被垃圾回收。

![image-20221006164335144](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061643192.png)

##### RemixDB 文件结构

Figure 6展示RemixDB的table file结构。每个data block 4KB，超过4kb的kv pair单独的占用一个大小为4KB整数倍的jumbo block。每个data block开头都有1个数组记录每个kv pair的偏移量。metadata块中是8比特的数组，记录每一个4KB data block中kv pair的数量，因此每个4KB data block中最多有255个kv pair，对于jumbo block，除了第一个4KB，其余空间在metadata blcok数组中记录为0，因此非0永远为1个data block的开头。使用metadata block和偏移数组可以快速访问相邻块和跳过一定数量的key。

![image-20221006165937768](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061659809.png)

Figure 7展示RemixDB中REMIX 文件格式。anchor key被组织成1个immutable B+-tree-like index，每个anchor key对应1个segmentID，以此确定cursor offsets和run selectors。每个cursor offset包含16bit的bllock index和bit的key index(Figure1中cursor offset为key在1个run中的下标，在物理存储上同1个run中的key可能存在不同block中)。 1个key的多个版本可能存储在多个table中，范围查询必须跳过旧version返回新version，在REMIX中，1个key的多个版本从新到旧排列，**run selector的最高位用来区分新旧版本**，因此顺序查找时检查run selector的最高位即可跳过旧版本。1个key的多个版本可能分布在2个segment中***(anchor key的稀疏索引二分查询时找到2个segment？，就需要分别检查2个segment的run selector？)***，因此搜索时需检查2个segment来找到最新的版本。为简化该场景，作者通过在构建REMIX时插入特殊特殊run selector作为占位符将所有版本移动到第2个segment，同时确保segment中key数量大于1个REMIX索引的sorted run数量即D>=H，使得1个segment中的足够保存所有1个版本的所有key(RemixDB不同区之间没有overlap，1个key的多个版本都在同1个分区中，每个sorted run中可能包含1个key版本，因此1个key最多有H个版本)。RemixDB中run selector占8 bit，第8位和第7位(0x80 and 0x40)分别表示旧版本和已删除key，特殊值0x3f表示占位符。

![image-20221006171135439](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061711484.png)



##### compaction

在每个partition中，compaction线程根据新数据的大小和现有table的布局评估compaction代价，随后执行以下4种程序：Abort、Minor compaction、major compaction、split compaction。1. Abort。当partition中新的table file很小，重写REMIX将导致较高的IO代价，为应对该种情况RemixDB可以在IO代价大于某个阈值的情况下放弃对1个partition的compaction，新数据保留在Memtable直到下一次compaction。极端情况为了避免绝大部分的新table file都过小导致大部分compaction都被aborted，作者进一步限制留在Memtable和WAL的新数据不超过memtable大小的15%，在这个限制的前提下IO代价过大的compaction才会被aborted。2. Minor compaction。如Figure 8所示，Minor compaction在不重写已有table file的情况下将新数据从Memtable写到1个partition中并重建REMIX。Minior compaction在compaction后table file数量(原有table file数量+新table file数量)**不超过阈值T**的情况下使用。3. major compaction。当预估的table files数量大于阈值T时，major compaction将现有table file 排序合并为数量更少的table file，使得后续可能使用Minor compaction。major compaction的效率由输入和输出table files的比值评估，RemixDB自动选择比率最小的合并方式；4. split compaction；major compaction在input/output比值较大如10/9时并不一定可以高效减少partition中table file的数量，在这种场景下，split compaction将所有的旧table和新数据进行sort-merge，合并结果被分割成多个partition来大量减少每个partition中table的数量。为避免分割成过多的小partition，compaction线程在第1个新partition中创建M个新table file后才转到下一个新partition。

![image-20221006185714153](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210061857204.png)

##### Rebuilding REMIXes

tiered compaction策略可以最小化放大写，但重建REMIX时去读已经存在的table需要较多的IO，因此作者使用1种高效的合并算法最小化创建REMIX时的IO开销。重建过程可以看作将两个新旧sorted run进行排序合并。作者借鉴generalized binary merging algorithm中通过大小比值来估计合并点，并在合并点附近寻找的思想，在RemixDB中使用anchor key确定包含合并点的目标segment，并在该segment中使用二分法查找合并点。使用这种方法，因为anchor key存在REMIX中所以访问anchor key并不消耗IO，在segment中使用二分法最多读取llogD的key(前面所讲通过blk-id和key-id读取)，最后新REMIX创建anchor key只需要访问每个segment中的1个key。REMIX的重建实际上是通过用现有table的读IO换取写放大和未来读取性能的提升。

### 4. Evalueation

##### 4.1 REMIX性能评估

1. 每次实验设计H个table(1<=H<=16)，每个table表又64MB个kv pair,其中key16B，value10B。实验中kv pair在table中的分配方式有2种：1是弱局部性，每个key被随机存储到1个table种；2是强局部性，每64个逻辑连续的key被存储到一个table中。实验将REMIX-indexed table与传统SSTable对比，SSTable中有10bits/key的布隆过滤器。在上述配置下，我们使用3种方式测试单线程下的吞吐量：1是seek；2是seek+next50，在单点查询后取回后边连续的50个kv pair，对于REMIX，设置segment大小为D=32，分别测试其有无in-segment的二分查找下性能，分为称它为满二分和部分二分查找；3是GET(点查询），测试SSTable在有无布隆过滤器下的性能。

2. Figure 11a弱全局性下使用REMIX和普通merging iterator下的吞吐量。当table文件数为1时，使用merging iterator的吞吐量比使用满二分法的REMIX高20%左右。因为该场景下二分法比较的次数一样(只有1个tableREMIX只需要segment内部二分)，但REMIX需要在二分查找是计算每个key的occurrence，且访问1个key时需要从头开始遍历，因此性能更差。 随着文件数量的增多merging iterator的吞吐量急剧下降，这是因为merging iterator需要在***每个table文件中都做一次二分查找***；同样，由于数据量增多使得key比较次数增大和更多次数的内存访问，REMIX性能也有所下降，但使用满二分法的REMIX相对于merging iterator在table文件为8和16时分别提升了5.1倍和9.3倍。 使用部分二分查询下的REMIX相比于满二分查询的REMIX性能下降20%-33%，因为部分二分查询需要在segment种遍历去查询目标key，平均查询次数为D/2，尽管这样其性能仍比merging iterator好。

   Figure 11b展示弱全局性下部分seek+next50的的性能。因为数据复制的开销，整体挺能相比于seek下降不少。merging iterator的性能较差因为其每一次next操作都需要读入多个key并比较来找到目标key。相反的REMIX在next操作种不需要比较任何key。 与Figure 11a先比，seek+next50实验中使用满二分和部分二分查询的REMIX在性能上几乎没有差异，这主要因为1. next操作占据主要的执行时间；2.部分二分查询下，REMIX的线性读取使得blocks读入到cache中，使得之后的连续读取更快。

   Figure 11c展示弱全局性下点查询的实验结果。REMIX对应的弧线比Figure 11a中对应弧线略低，因为GET操作在seek操作的基础上还需读取目标kv pair。在带有布隆过滤器的SSTable性能搜索优于在REMIX-indexed table上搜索，这是因为1. 使用布隆过过滤器可以在极小代价下定位到目标table；2. REMIX管理更多的key因此在1个SSTable上的搜索比在1个REMIX上搜索更快。

   * ***merging iterator是啥？？*** 在带有布隆过滤器的SSTable上时怎么搜索的？？a和c有什么关系？![image-20221005145320147](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051453198.png)

3. Figure12展示强局部性范围查询和点查询的性能表现，其中Figure 12a和12b中整体趋势与Figure11相对应曲线的整体曲线类似。通常来讲，强局部性可以使二分查找更快因为最后几个比较的key通常在1个block中，而merging iterator在强局部性下的性能依旧较低因为其密集的key比较占据主要时间。使用部分二分查询的REMIX性能提升比满二分法的REMIX性能提升更多，因为数据局部性的提升减少在目标segment上扫描的开销，缓存miss的情况极少。 对于Figure12C，强局部性加快了seek操作的时间，使REMIX点查询性能提升；然而带布隆过滤器的SSTable的搜索性能几乎不变，这是因为搜索的主要开销由假阳性率和在单个table中的搜索时间决定(考虑没有假阳性的情况，强局部性只加快了搜索1个table的时间)。![image-20221005154937046](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051549104.png)

4. Figure 13展示8个table file下不同segment大小D下的性能表现。对于**seek only**操作，当采用**部分二分**时，不同D值下的系统性能差异较大，这是因为当D值增大时，线性扫描的开销也会随之增大；相反，使用满二分查找下不同D值对系统系能的影响较小。同时较大的segment大小导致更慢的随机访问速度，导致额外开销增大性能降低。对于seek+next50操作，数据复制占据主要时间因此不同D值对系统吞吐量影响不大。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051620452.png" alt="image-20221005162016407" style="zoom: 67%;" />

##### 4.2 RemixDB性能评估

1. 作者将RemixDB与LevelDB、RocksDB和PebblesDB比较。Level支持1个compaction线程，RocksDB和RemixDB在实验中使用4个compaction线程。RemixDB、LevelDB和RocksDB都配置为64MB table file。PebblesDB使用默认配置。对于所有kv-store，取消压缩并设置4GB 的块缓存。实验中设置vlue大小为40，120，400字节三种大小，使用16字节固定长度的key值。

2. **探究不同kv 大小和数据加载方式对范围查询的影响**。每次实验作者先加载100 million kv pair到数据库中，随后使用sequential、Zipfian、uniform3种访问方式和4个线程测试吞吐量。Figure 14中可以看出，不同访问方式下，不同value大小对应的吞吐量结果大体趋势相同，RemixDB吞吐量最高，LevelDB比RocksDB和PebblesDB块至少2倍。 sequential加载使得不同table files中不会有key 重叠，因此1个seek操作只需要访问1个table file。LevelDB和RocksDB中L0层的每一个table都是一个sorted run，但其他level只有1个sorted run；PebblesDB允许每一层有多个sorted run。LevelDB比RocksDB快2倍以上因为RocksDB在L0层保持8个table，并没在sequential load时将他们是送到更深层次，而LevelDB直接将table push'到L2或L3如果没有overlap 的话，因此LevelDB在sequential load场景下L0永远是空的，因此当seek操作时RocksDB至少需要sort merge12 sorted runs然而LevelDB只需要合并3-4个sorted runs。数据局部性极大影响seek性能。Figure14每个实验中，使用uniform加载方式的吞吐量比对应的使用Sequential方式的吞吐量低50%。同时，sequential方式下系统性能对value大小并不敏感，这是因为内存的复制开销并不大。

   ![image-20221005164107158](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051641203.png)

   **探究不同存储大小和query长度对范围查询的影响。**每一次实验按随机顺序加载固定大小的kv的数据集到一个数据库中，最后使用4个线程和Zipfian方式范围查询。Figire15 展示不同存储大小下不同数据库的吞吐量情况，RemixDB性能优于其他数据库，然而性能随着范围查询长度的增加而下降，这是因为更长的范围查询需要更多预取更多的数据。当store大小增大到256GB，LevelDB的性能急剧下降到和RocksDB一个相同水平，这是因为cache miss导致的密集IO占据了主要的执行时间。

   ![image-20221005171530369](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051715411.png)

4.  **探究写性能**。首先通过随机插入1个256GB的KV数据集到空store中进行测试，该数据集有2billion个kv pair，value大小为120B。Figure16所示，采用tiered compaction策略的RemixDB和PebblesDB写吞吐量相对较高，总的写 IO分别为1.5T和2.37T，对应的写放大系数为4.88和9.26。采用leveled compaction策略的LevelDB和RocksDB写放大则较高分别为16.1和25.6。RocksDB和RemixDB有较大的**读IO**，RocksDB有4个compaction线程，block和page的缓存效率较低，因此有更多的读IO。尽管RemixDB比RocksDB读IO更多，RemixDB总的IO数少于RocksDB。总而言之，RemixDB以增加读取IO为代价实现了低写放大和高写入吞吐量。

   ![image-20221005184355865](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051843909.png)

5. 接下来评估RemixDB 在具有不同空间局部性的工作负载下的写入性能。作者采用三种访问模式：sequential、Zipfian和Zipfian-composite。对于每种访问模式，实验先随机写入256GB数据随后进行20亿次更新，最后在更新阶段测量吞吐量和总IO。如Figure 17所示，Sequential负载吞吐量最高因为每次compaction只涉及到少量的连续partition，其中写IO主要是写日志和写回新的table files，大约为用户写的2倍，读IO大约等于和现有数据大小。另外两个workload因为在Memtable中重复写所以减少了写IO，然而这两个负载较为分散的更新导致Memtable中的更新变慢和更多的partition被压缩。

   ![image-20221005191027118](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210051910146.png)
   
   

5. 作者使用256GB的存储并用4个线程运行YCSB A-F负载。如Figure 18所示，RemixDB除了workload D外表现都优于其他数据库，woload D中大量读取近期写入的少量数据，具有高局部性，大多数请求直接从Memtable中获得。尽管Figure 11c中展示在micro-benchmark中，Remix表现不如布隆过滤器，但RemixDB在workloadB、C两个主要为点查询的workload表现优于其他数据库。

   ![image-20221006212929391](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210062129434.png)

![image-20221006213809352](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210062138386.png)



### 总结

本文介绍了REMIX，一种针对LSM-tree下范围查询设计的紧凑多表索引数据结构。其主要思想是记录多个table file的全局排序视图，使用稀疏索引和二分查询的思想快速定位，并根据全局排序视图进行key的连续扫描。通过REMIX，采用tiered compaction策略的LSM-tree可以在保持较低写放大的效果下提高范围查询的性能。