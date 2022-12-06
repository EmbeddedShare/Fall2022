### YourSQL: A High-Performance Database System Leveraging In-Storage Computing

这是三星公司在2016年发表于VLDB的一篇工作。作者实现支持in-storage computing数据库架构YourSQL，将SSD中完成early filtering操作，减少数据库服务器和硬盘之间的数据传输量，避免加快数据密集型查询请求进行全表扫描，减少请求的响应时间。并在该访问的基础上提出了software filtering和hardware filtering协同，批量读取等优化，实验结果显示YourSQL有着明显的性能提升。

#### Motivation

​	以往的Early filtering工作带来的性能提升依赖于选择性selectivity，指的是选中行占数据集所有行的比率，往往认为更低的选择率会带来更大的性能提升。然而实际上基于行的选择率与IO数量没有特定关系，这取决于被选中行的分布情况。因此作者定义了基于page页的选择率filtering ratio。如Table 1，作者通过在Mysql中进行基于ICP(Index Condition Pushdown)的early filtering来证明能有效提供数据密集型query的性能。![image-20221205162115931](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212051621156.png)

​	受上述ISC对性能提升的启发，作者设计了基于ISC的数据库架构YourSQL，Figure 1展示了YourSQL与传统架构的异同。YourSQL配备了ISC-enabled SSDs，整体软件栈通过ISC module感知SSD。主要设计有 1. 下放到ISC SSD计算的任务需要从query操作中提取并封装成ISC task下放到ISC SSD；2. 数据库系统到ISC SSD之间的接口以满足ISC task的下放和结果数据的提取；3. 修改Query Planner以支持针对ISC-enabled数据库系统优化的算法；4. 相比于将所有数据交由host处理，ISC SSD可以仅将相关的数据页列表交给host，host再通过批量读取技术读入相关页。

![ ](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212051635271.png)

#### DESIGN AND IMPLEMENTATION

对于query请求，YourSQL完成如下工作：1.解析；2. 选择early filtering候选表；3. ISC sampler评估减少的IO数量；4. ISC sampler决定候选表是否为early filtering目标表，并指定query plan； 5. 存储引擎调用ISC filters完成early filtering；6. host通过批量读取等优化技术读取结果![image-20221205193256695](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212051932755.png)

##### Basic Design

1. Early filtering目标表的选择和join顺序

​		在Mysql中，ICP是否促发取决于是否有二级索引，这并不能保证减少IO次数。YourSQL则依赖评估的filtering ratio，将filtering ratio足够高的表作为early filtering的目标表。评估filtering ratio不可能通过遍历扫描计数满足条件的表，开销过大，作者两个指标评估：先通过1.limiting score(简单的启发规则，开销可以忽略不计，与filtering ratio没有定量关系)选择early filtering候选表，在使用2. estimated filtering ratio确定其是否为目标表。筛选条件越严格的请求limiting score越高，与谓词数量、谓词类型、表行数有关，请求中的每个谓词有一个对应的score，limiting score等于这些谓词对应score的总和。early filtering目标表总被放在join顺序中的第一位。Table为YourSQL对Query 2的join顺序。

![image-20221205201338840](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212052013887.png)

2. 谓词下推

   如Figure 4所示，YourSQL采用软硬件两级过滤器，硬件filter位于ssd中，采用pattern matcher，可以大量过滤不相关数据，但存在假阳性，因此引入软件filter进一步过滤。过滤的结果是一个字节数组称为match hints，其中每一个元素对应一个page页，当该page页符合条件则该元素置1，最终结果match array逐级向上传递。

   硬件pattern matcher采用最多3个16字节的二进制key进行字节粒度的匹配，因此YourSQL首先将谓词转为二进制形式，字符串转为ascii码，整形数字用4字节二进制表示。

   ![image-20221205221544273](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212052215315.png)

3. 通过Match Hints获取相关table

   如Figure 5所示，YourSQL的1个处理1个请求主要包含3个步骤：early filtering，匹配页的读取和行处理。如Figure b所示，其中host完成的页读取和行处理可以和ssd完成的early filtering并行执行。ISC SSD的Early filtering每次传回1个iteration unit单元的match hints，存储引擎根据match hints到SSD中依次读取page并处理。
   
   ![image-20221205225842437](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212052258498.png)

##### Optimazation

1.  采样驱动的谓词下推

   由于limiting score 和filtering ratio没有定量关系，因此在ISC SSD中引入sampler，filtering ratio由sampler得到的匹配的page个数除以总page个数，当filtering ratio高于某个阈值触发FCP。为了确定该阈值，作者将query分成了4类：1. 单表查询；2. 没有子查询的join查询；3. 带有派生表的查询(派生表只来自From子句)；4. 带有子查询的查询。对于类别1，当过滤后page数目小于全部page的30%，random IO读取过滤后的page的时间就会小于顺序读取全部page的时间，因此设置阈值为0.3。对于类别3，阈值简单设置为1，因为减少中间引用表的大小可以加速query。类别2和4的阈值需根据情况实时计算，需要构建一个精细化的成本模型，作者目前先将阈值暂定为0.7。

2. software filtering

   由于硬件filter的pattern matcher只能采用字节粒度的匹配，因此存在假阳性的情况，因此引入软件filter。软件filter可以更加准确的过滤，但是由于频繁的访问SSD开销也大，因此需要平衡过滤精度和开销，在整体上提高系统性能。

3. 批量读取

   YourSQL采用2中批量读取技术。1. bulk read，对match hints中的page进行批量读取。2. aggressive prefetch，得益于更高的内部带宽，ISC filter获得match hints的时间比bulk read的时间更短，因此在bulk read的同时可以进行对下一个match hint的预读取。如Figure5 c所示。

#### Evaluation

1. 实验环境

![image-20221206200739433](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062007570.png)

2. Query 1简单查询，只用到硬件filter。![image-20221206201128035](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062011073.png)

3. Query 2join查询

   ![image-20221206201213487](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062012519.png)

4. 不同的内存大小的影响。内存越大加速效果约不明显，这是因为Mysql性能提升更加依赖于内存大小。

![image-20221206201332848](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062013886.png)

5. 采用TPC-H测试的结果，其中支持FCP的8个query平均性能提升15倍。![image-20221206201748297](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062017352.png)

6. 测试上诉优化方案。![image-20221206201827349](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062018387.png)![image-20221206201822367](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062018409.png)

7. 能源消耗。由于YourSQL用满了SSD的带宽，因此在前期消耗更多能源，总体上MySQL消耗的总能量是YourSQL的24倍。

8. ![image-20221206202018018](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062020064.png)

   ![image-20221206202025321](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212062020358.png)

   









