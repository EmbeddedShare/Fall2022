### HotRing: A Hotspot-Aware In-Memory Key-Value Store

Alibaba Group

------

这是阿里在fast20上发表的1篇文章。作者对用作缓存的in-memory kvs的访问分布进行分析，发现in-memory kvs也存在这热点访问的问题，即大规模的访问只请求小规模的数据。但现有的in-memory kvs的实现都没有考虑到热点数据的感知问题，而是将数据均匀的分布，这导致了对热数据请求的处理与冷数据请求处理的方式一样，造成大量不必要的内存访问开销。在此基础上作者提HotRing，1个具有热点感知能力的in-memory kvs，在链式结构上根据热点数据位置放置头指针，使得头指针访问热数据需要的内存访问代价最小。并使用Random movement 策略和Statistic sampling 策略解决了热点变换问题，并使用lock-free的设计解决了并发访问的问题。实验表明HotRing对比其他in-memory kvs性能提升了2.58倍。

### Background

1. Figure 2展示典型的hash index结构，包括1个hash table和多个collision chain，访问某个key时需先进行hash定位到hash table 的1个entry，后在collision chain查找该key，为了避免key值比较的开销，将hash value分为两部分：1.hash table部分用于定位hash table 的entry；2. 包含在每个Item中的tag部分避免长key值的比较。可以看出，hash索引并不了解热点数据(均分分布)，位于collision chain尾部的hot item的获取需要更多次的内存访问，而在倾斜工作负载中，对hot-item访问开销的一点点增大都会导致严重的性能下降。

   ![image-20221011143758746](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210111518497.png)

2. 下面评估热点感知潜在的好处。在传统chain-based hash index中，假设共有N个item，存放在B个bucket chain中，则每个bucket chain的**平均长度L = N / B**，则获取1个item所需的内存访问次数为$E_{\text {chain }}=1+\frac{L}{2}=1+\frac{N}{2 \cdot B}$，其中1为访问哈希表。对于理想的热点感知的hash index，内存访问次数应与该item的热度相关，最热的item需要最少的内存访问次数。作者对Zipfian分布中的热度建模，第X热的item访问频率表示为$f(x)=\frac{\frac{1}{x^{\theta}}}{\sum_{n=1}^{N} \frac{1}{n^{\theta}}}$，其中${\theta}$表示skewness factor。假设热点数据均匀分布在B个bucket chain中(每个bucket包含1个top B热点数据，1个topB+1到top2B热点数据)，则访问1个item需要的内存访问次数为

   $E_{\text {ideal }} = 1+\sum_{k = 1}^{L} F(k) \cdot k \\ = 1+\sum_{k = 1}^{\frac{N}{B}}\left[\sum_{i = (k-1) \cdot B+1}^{k \cdot B} f(i)\right] \cdot k$,其中F(k)表示各条chain中第k热的item 的频率之和。Figure3对比了1次查询下，传统hash索引和理想热点感知索引下访问内存的的次数。

   ![image-20221011153338926](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210111533986.png)

3. 热点感知索引存在如下挑战。1. 热点变化。需要1种轻量化的方式追踪热点变化；2. 每个热点数据都被大量地并发读写，因此需要支持高并发的读写操作以维持性能。对于热点变化问题，作者的设计原则是避免对chain上的item重新排序，而是移动头指针指向最热或全局更优的item。为解决由于头指针移动而导致部分item不被访问的问题，作者采用ordered-ring结构替代collision chain，但这种结构无法达到理想下的热点感知。对于并发问题，作者采用lock-free来消除加锁和同步的开销。

### Motivation

为解决面向磁盘的热点访问问题，In-memory kvs被广泛用于缓存热数据，但是in-memory KVS的热点访问反而被忽视了。如Figure 1所示是阿里巴巴产品环境的in-memory kvs的访问分布，其显示在日常和极端情况分别由50%和90%以上的访问时针对1%的数据，这说明了in-memory kvs的热点访问问题的严重性。有很多的数据结构如调表、哈希都可以实现KVS，但作者发现绝大部分的KVS都不具备热点感知功能，他们不考虑数据的访问频率而是将数据均匀的分布。

![image-20221013161128622](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131611663.png)

### Design of HotRing

#### Ordered-Ring Hash Index

Figure 4展示了HotRing索引结构，其将传统的collision chain改进为collision ring。

![image-20221012101846014](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210121018074.png)

该结构存在1个问题去解决：当目标item未找到会导致head指针的无限循环。通过标记head指针的起点item的方案是不可取的因为起点item可能被其他的query请求修改或者删除。因此作者采用了有序环结构，当head指针连续读到两个item分别小于和大于目标item时即停止扫描。同时采用为避免较长key比较的开销，采用了上文提到的tag位，item通过$\operatorname{order}_{k}=\left(\operatorname{tag}_{k}, \text { key }_{k}\right)$来排序。假设当前访问item i，目标item为item k，则有以下情况。假设1个环包含n个item，则查找1个item平均需要比较(n/2)+1次。Figure 5展示1个实例。

- 命中：$\operatorname{order}_{i}=\text { order }_{k}$
- 未命中：
  - $\text { order }_{i-1}<\text { order }_{k}<\text { order }_{i}$
  - $\text { order }_{k}<\text { order }_{i}<\text { order }_{i-1}$
  - $\text { order }_{i}<\text { order }_{i-1}<\text { order }_{k}$

![image-20221012101030810](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210121011642.png)

#### Hotspot Shift Identification

术语:

* hot item: 头指针指向的第一个item
* cold item：其他未cold item
* hot access：对hot item的访问
* cold access：对cold item的访问

我们可以通过将头指针指向热点item来提高查询性能。对于热点item的识别与确定，我们考虑2方面：识别的准确性和延迟，识别的准确性由识别出的热点item的比例衡量，反映延迟由1个新热点数据的产生到探测到该热点数据间的时间衡量。鉴于此，作者采用2种方式：1. random movement strategy，2statistical sampling strategy。

**Random movement strategy**反应延迟较低但准确度也低。主要思想在不记录任何历史元数据的情况下，头指针阶段性的移动到1个可能的热点数据。每一个线程被分配一个线程专属的参数用来记录该线程执行请求的次数，每经历过R次请求后，当第R次请求是hot access，头指针指向不变，否则头指针移动到由cold 访问的item，成为新的hot item。R的确定影响反应延迟和识别准度，当R较小时，反应延迟较小但可能导致头指针频繁和低效的移动。在作者的应用场景中，数据访问高度倾斜，因此头指针的移动并不频繁。*在测试中，作者设置每个bucket(ordered ring index)种item各位为5-10，其中1个item为hot item，因此作者将R经验性的设置为5.* 当工作负载倾斜程度较小时(每个item访问的很均匀，会频繁变更hot item)，random movement strategy将很低效，而且该方法无法适用于有多个hot item的情况(头指针频繁的在多个hot item种转移)。

**Statistic sampling strategy**。如Figure 6(a)所示，每个头指针包含3部分：active bit用来控制抽样统计、total counter(该ring的访问次数)、address。如Figure6(b)所示是1个item的数据结构，其中rehash位用来控制rehash进程*(后续会提)*，occupied位用来确保并发正确性*(后续会提)*，用剩下14位记录该item的访问次数。

哈希表通常很大，持续并发的更新大量ring上的统计信息会导致严重的性能下降。作者采用阶段性采样的方式，每一个线程本地的计数器记录该线程执行请求的次数，每R个请求完成后判断是开展下一轮信息统计(根据是否置active flag为1)，当第R次访问为一次hot access表示当前热点的确认是准确的不需要开展下一次信息统计，否则进行下一轮统计。当hot access被设置以后(应该是开展新一轮统计的意思)，接下来的访问会被记录在total counter和counter中。采样统计的过程需要额外的compare and swap操作，导致暂时的访问不足，因此为缩短这个过程作者设置样本的数量和每个ring中item的数量相同。

![image-20221012145720867](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210121457915.png)

采样统计完成后，根据统计结果进行hotspot的调整并重新设置头指针，这些操作由采样统计的最后一个线程完成。首先使用CAS原语将Active bit置为0，表示非统计过程；接着计算每一个item的访问频率，表示为$n_{k} / N$,其中N为该ring的总访问次数，$n_{k} $为第k个item的访问次数；接着计算当头指针指向item t时的收益，表示为$W_{t}=\sum_{i=1}^{k} \frac{n_{i}}{N} *[(i-t) \bmod k]$，表示头指针从item t出发遍历长度的加权平均值。其中$[(i-t) \bmod k]$表示头指针从item t移动到item i所需的步数(k好像是该ring的item总数)。因此选择具有最小的$W_{t}$作为hot item可以使得整体的热点数据访问更快，使用CAS原语将头指针从原来的位置移动到新的hot item。注意这种策略可以处理同时具备多个热点数据的情况，并选择最合适的位置**(不一定是最热的item)**避免头指针在多个热点数据之间反复移动。

对于value小于8比特的item的更新，HotRing采用就地更新的方法(机器最多只支持8比特的原子操作)。而对于较大value的item，只能采用read-copy-update(RCU)的方式更新，如Figure 7所示。在这种情况下，被更新item的前一个item也需要被修改来指向更新后的item，因此如果是头指针指向的item被更新，则需要遍历整个ring来找到上一个item，也就是说写密集型的hot item使其前一个item也变热，因此作者对于1个itemRCU更新，前一个item的count也加1(帮助下一次hot item的识别)。

当对head item更新或删除时，头指针需要更新，若头指针随机指向1个cold item则会影响系统性能。因此作者采用如下策略：1. 对于RCU更新，最近更新的item有最大的可能性被再次访问，因此头指针移动到head item(被修改item)的新的item；2.对于删除操作，头指针直接移动到下一个item。

![image-20221012165304118](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210121653175.png)



#### Concurrent Operations

头指针的移动使得无锁设计更加的复杂，主要体现在2个方面：1. 头指针的移动可能与其他操作线程并发(如删除item的线程)，需要防止头指针指向无效的item；2. 当删除或修改头指针指向的节点时需正确高效的移动头指针。**对于读**操作，不需要额外的操作来确保并行性。**对于插入**操作，如Figure 8(a)所示，需要修改前一个结点的Next Item Address，两个并发的插入操作可能竞争同一个Next Item Address，CAS确保只有一个线程成功。**对于更新**操作，当value值小于8字节的原子性就地更新不会影响其他操作，对于RCU更新，HotRing采用Occupied bit来确保并行性，以Figure 8(a)为例，update过程分为2步：1. item B的Next Item Address被设置为Occupied状态，此时item C的插入就会失败；2. item A的Next Item Address自动设置为B‘，B’的Occupied位置0。**对于删除操作**，需要确保被删除item的Next Item Address在删除过程中不被修改。以Figure 8(C)为例，在删除item B时将B的Occupied置为1，则Item D的修改操作无法修改Item B的Next Itam Address，直到B删除完成。**对于头指针的移动**,需要额外的操作来确保头指针移动和其他操作并发的正确性，有3类情况：1. 热点数据识别导致的头指针移动。使用Occupied位确保在头指针移动的过程中该item不会被修改或删除；2. head item更新导致的头指针移动。设置新版本head item的Occupied位来确保他在头指针移动过程中不会被更新或删除；3. head item删除导致的头节点移动。要对即将删除的item和他的下一个item设置Occupied位，确保要删除结点的Next Item Address不被修改和下一个节点被删除或修改使得头节点指向一个无效的item。

![image-20221012191535339](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210121915388.png)

#### Lock-free Rehash

数据的插入使得每个ring中的item数量激增，使得每次访问都需要遍历更多的item导致性能下降，考虑到hotspot item的存在，HotRing使用访问开销来触发一次resh，具体步骤如下：**1. 初始化。**HotRing创建一个rehash线程，该线程初始化一个大小为原来两倍的hash表，原hash table中每个bucket的1个头指针分裂为2个头指针(2个桶)，原有hash value中需要k位用来作哈希表定位，则现在需要k+1位，对应的tag变成n-k-1位。原有tag的范围位[0,T)(T=$2^{(n-k)}$)，则现两个新的头指针管理的tag范围分别为[0, T /2)和[T /2, T )。同时如Figure 9(b)，rehash线程创建1个包含2个rehash item的rehash结点，两个rehash item对应两个新的头指针，每个rehash item除了不真正具备kv pair外与其他item完全相同，其Rehash位置1来标志其位rehash item，rehash item的tag被分别设置为0和T/2(同一个sorted ring中用tag排序，分别为两个新rin给的最小tag)。**2. 分裂。**如Figure 9(C)所示，该阶段将2个rehash item插入到环当中，至此新的hash表进入活跃状态，后续的访问操作由新hash table提供服务，先前对旧hash table的访问操作可以判断Rehash位继续进行。至此分裂完成，对一个item的查找至多只要扫面先前一半数量的item。**3. 删除。**如Figure 9(d)所示，rehash线程将rehash node删除。在此之前rehash线程需经历一个过渡期确保所有对旧hash table的访问都已结束。

![image-20221013091740098](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210130917433.png)

### Evaluation

#### 实验环境

硬件环境如Table 1所示。

![image-20221013104525470](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131045530.png)

key大小设置为8字节，value设置为8和100字节分贝对应就地更新和RCU更新。Table 2显示不同zipfian分布的参数$\theta$不同热点数据定义下热点数据的访问比例，其中$\theta$为[0.9, 0.99]代表日常场景，[1, 1.22]表示极端情况。实验将HotRing与Chaining Hash(作者实现的lock-free链式索引结构)和FASTER、Masstree、Memcached等kvs对比。考虑到索引大小的影响，实验所采用的对比组的各种索引大小相同。

![image-20221013105816884](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131058945.png)



#### 与现有系统的比较

Figure 10展示HotRing和其他方法在多个工作负载下的表现，其中HotRing-r表示采用random movement strategy的HotRing，HotRing表示采用sampling statistics strategy的HotRing。对比其他系统，HotRing在各个负载下的性能表现都更加优秀。对于有大量插入操作的D和F负载，HotRing的表现依旧优秀，比其他系统提升1.81-6.46倍。

![image-20221013110604858](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131106895.png)·

如Figure 11(a)所示，下面探究不同collision chaining长度下各个系统的性能表现。作者调整key-bucket ratio(每个bucket具有的item数量)为2-16。当key bucket ration为2时chainiing hash和FASTER还有较好的性能表现，这是因为当collision chain较短的时候，访问hot item的内存访问开销相对较小，当collision chain较长时，对热点数据的频繁访问导致的内存访问开销使其性能急速下降。然而HotRing性能始终较好，这是由于HotRing将热点数据放在头指针的附近，对热点数据的访问可以减少很多不必要的内存访问。 

Figure11(b)展示不同HotRing和其他方法在不同的zipfian参数$\theta$下的吞吐量($\theta$越大表示越集中访问热点数据)。随着$\theta$的增大chaining hash和FASTER的性能提升并不明显，因此这两种方式对热点数据并没有感知。相反，HotRing随着$\theta$的增大性能提升显著。即使在$\theta$小于0.8时即访问数据倾斜程度较小时，HotRing-s也有更好的性能表现因为其可以在有多个热点之间选择最佳的头指针位置。

![image-20221013142401990](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131424036.png)

Figure 12展示包含RCU操作下不同方式的吞吐量。作者使用100字节value的写密集型负载来测试。其中HotRing-s(w/o)表示未对RCU操作设置处理策略的HotRing(专用策略上文有介绍)。由图可以看出未指定处理RCU策略的HotRing-s(w/o)性能极差因为其每对1个item的修改都需要遍历整个ring来找到上一个item。当key-bucket ratio较小时，HotRing-s性能比FASTER略差，这是因为hotring需要额外的内存访问和Occupied 位上CAS操作来完成RCU操作，但这些开销随着每个ring的大小增大被缓解了。

![image-20221013144413107](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131444157.png)



#### 细节设计调查

该部分探究热点感知相比传统链式哈希的优势。

Figure 13展示对HotRing和Chaining Hash执行开销的分解。其中HeadPointer表示定位头指针的开销，HeadItem表示访问head item的开销，Non-HeadItem表示访问非头节点的开销，HashValue表示哈希计算的开销，Benchmark表示读入和执行工作负载的开销，Other表示内核开销。可以看出，Chaining Hash主要是非head item访问的开销，这表明热点数据在Chaining Hash方式下时均匀分布的。相反，HotRing极大的减少了Non-HeadItem的开销，尤其是HotRing-s因为其对热点感知的精确度更高。

![image-20221013151157372](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131511416.png)

Figure 14显示在热点变化之后吞吐率的变化情况。可以看出HotRing-r对热点变化的感知是最快的，但是其顶峰吞吐率并非最高因为其热点识别的准确度相对较低。

![image-20221013152325443](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131523485.png)

由Figure 15a所示，对于read miss，HotRing和Chaining Hash之间的性能差异越来越大，这是因为对于sorted ring结构，item的平均制度只有链表长度的一半。

R表示每R个请求过后重新评估热点数据并移动头指针。当R较小时，热点变换时的热点感知延迟会较小，但也会引起很多不必要的头指针移动。由Figure 15(b)所示，当R太小或太大时性能都有轻微的下降。

![image-20221013152601939](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131526980.png)

HotRing-s采样统计的最后1个线程需要计算item的访问频率来确定头指针的最佳位置，此过程可能存在延迟。Figure 16展示了10万次访问的延迟分布，当$\theta$=1.22时，99%访问的延迟都在2us以内，但最高的长尾延迟达到8.8us，相同的$\theta$=0.99时，99%的访问延迟在3us内，但长尾延迟达到9.6us。

![image-20221013154211160](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131542202.png)

Figure 展示了在进行rehash时HotRing的性能表现，I、T、S分别表示initialization, transition和splitting 阶段，rehash操作使得HotRing在数据增长的情况下保持吞吐量，短暂的性能下降是由于新hash表创建时缺少热点感知。

![image-20221013155350423](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210131553462.png)



### 总结

对于目前被广泛使用的KV Store，热点感知问题变得越来越严峻。因此作者探究了构建具有热点感知功能的in-memory KVS的机遇和挑战，并提出了针对大规模访问小部分数据问题的哈希索引结构HotRing。HotRing能够动态感知热点数据的变化，并在大部分场景下通过2次内存访问就可以取得热点数据；同时HotRing彻底地使用了无锁设计。实验表明，相比于其他方法，HotRing可以达到2.58倍的性能提升，并已经作为阿里巴巴产品Tair的一个组件被应用。









