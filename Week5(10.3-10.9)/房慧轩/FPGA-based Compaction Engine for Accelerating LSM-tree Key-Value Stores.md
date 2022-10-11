# **FPGA-based Compaction Engine for Accelerating LSM-tree Key-Value Stores**
Author : Xuan Sun, Jinghuan Yu, Zimeng Zhou and Chun Jason Xue

Research unit :
Department of Computer Science
City University of Hong Kong
{xuansun-c, jinghuayu2-c, zmzhou4-c}@my.cityu.edu.hk, jasonxue@cityu.edu.hk

以KV存储为基础的LSM-Tree在写方面的性能表现很好，因此其应用也越来越广泛。在LSM-Tree中，合并是影响吞吐量和整个系统性能的一项关键因素。这篇文章选择将合并操作从宿主机CPU卸载到FPGA上进行。同时为了能高效利用FPGA，还设计了多个优化策略。最终实验结果表明，设计的FPGA-Based合并引擎相比于原本的CPU设计可以达到92倍的加速率，在随机写工作负载下也可以实现6.4倍的吞吐量提升。

**介绍**

LSM-Tree相比于传统的RDBMS，其在写操作方面具有更加优秀的性能，因为它可以将随机写转变为顺序写。已经有了部分关于提高LSM-Tree的性能的研究，但它们主要关注减少合并操作的发生频率以及推迟写暂停，合并的影响没有从根本上被解决。因此，本文使用FPGA来加速LSM-Tree的合并性能。而使用FPGA也面临了一系列的挑战：
1、FPGA的合并单元必须被设计到能确保FPGA的性能优于CPU的程度。
2、流水线设计应当由FPGA优化，以此来充分利用FPGA的优势。
3、硬件和软件之间的通信接口设计必须考虑到数据损坏的情况。
4、宿主端何时以及如何引入这一硬件加速模组也需要调查。
在考虑上述挑战的基础上，这篇文章主要做了以下工作：
1、设计了一个硬件软件相结合的LSM-Tree based architecture，FPGA在其中负责协同处理合并过程，来实现加速。
2、在FPGA上设计了一个可装载合并工作的引擎，其中包括解码器、比较器和译码器三个模块，除此之外，还提出了索引数据块分离、KV分离和完全利用FPGA带宽等优化策略。
3、软件方面与LevelDB相整合，设计了统一的输入输出内存接口，并将宿主端的合并任务调度器利用FPGA进行了优化。
4、进行实验，并验证了文章所提出的FPGA-based合并加速引擎的性能提升效果。

**背景**

​​![图1.装载了FPGA加速器的LSM-Tree based key-value store的系统架构](https://img-blog.csdnimg.cn/dd49410fff4040018295e9a5c1a76dd9.jpeg#pic_center)

如图1所示，传统的LSM-tree based KV存储由两种类型的任务组成——负责读写任务的主线程和负责调度以及执行合并操作的背景线程，当CPU被繁重的合并任务占满时，主线程就会被减缓。
在合并操作中，也有两种类型，一种是存储格式转换，一种是将不同SSTables中的数据合并。如图1所示，每层都容纳了一定数量的由固定大小的SSTables，当一层的SSTables的数量到达极限时，背景线程就要收集这一层和其下一层的相关SSTables并将其合并，生成的新的SSTables被放置在下一层。这种合并是系统性能的主要瓶颈，也是文中主要关注的内容。
SSTable是一种包含了索引块和KV数据块的文件，其中的数据块并不紧密相连，在扫描完成一个数据块后，需要从索引块中获取下一个数据块的位置及大小。

**系统总览**

如图1，文中设计在传统设计的基础上添加了一个负责执行合并任务的FPGA卡，其中包含一个FPGA芯片和一个DRAM。当第i层的SSTables数量达到阈值后，背景线程出发合并执行的过程如下：
1、CPU收集需要合并的SSTables的元数据。
2、背景线程计算所需要的输入数量，当i=0时，这个值与第0层的SSTable数量相同，否则等于1。
3、CPU在磁盘中分配为SSTable文件以及后续新产生的结果分配连续空间。
4、输入数据通过PCIe总线被传输给DRAM。
5、合并引擎从DRAM中读取数据，开始工作，过程中当输入数据大小到达FPGA芯片的容纳上限时，会将其传输给DRAM。当全部执行完毕后，CPU接收到信号，并将产生的结果放入内存。
6、CPU将结果写入磁盘，并执行收尾工作。

**硬件设计**

![图2.优化后的流水线合并过程](https://img-blog.csdnimg.cn/ca1c54340c9c4e1ca4c7a88245bdcc41.jpeg#pic_center)

硬件中主要有三个模块：**解码器、比较器**和**编码器**。
**解码器**的功能是从SSTable从提取KV对，当存在N个解码器时，最多支持N个输入同时进行。解码器读入的KV对会存于Key-Value-Buffer中等候比较器的下一步处理。
**比较器**负责选出key值最小的KV对并进行进一步的有效性检查。当它选出一个KV对后，会将相应的输入编号以及标识是否有效的Drop flag传给Key-Value Transfer模块。
**编码器**用于对选出的KV对进行snappy压缩操作，使得系统中的SSTable的存储格式一致。当一个数据块大小到达阈值时，这个数据块会被传入DRAM，同时相应数据也会被添加到索引块中，索引块存于BRAM（FPGA on-chip RAM）中，当一个SSTable的大小达到阈值时，这个索引块会被写入DRAM，同时编码器会重启，为下一个SSTable做准备。当整个过程完成后，这些数据会一同被传回宿主端。

在硬件设计中，使用了三种优化策略
1、在译码和编码过程中令索引块和数据块分离。由于SSTable的存储方式，若读取时只有一个读指针，那么就需要令指针在索引块和数据块之间多次移动，因此分离后可以加速，而且这也使得索引块不需要等待所有数据块完成后才写入内存，减少了传输时间，降低了BRAM的占用率。
2、Key和Value分离。较长的value数据会影响整个执行时间，而考虑到在比较操作中根本不需要value值，因此将其分离可以提高总体性能。

![图3.提升数据带宽](https://img-blog.csdnimg.cn/2a435025db0343608310520e081da537.jpeg#pic_center)

3、优化数据传输。如图3所示，为了充分利用FPGA的带宽，可以将数据块输入输出的带宽扩大，同时加入Stream Downsizer/UPsizer来平衡数据流。

**实验结果**

在两路输入的情况下

![图4.不同value length和V情况下的加速率](https://img-blog.csdnimg.cn/5b9d44fddae445cc8d387f9eab6004ff.jpeg#pic_center)

如图4所示，CPU和装载FPGA的两者的合并速度都会随着value length的增加而增加，同时V也会导致总的加速率的增长。而装载了FPGA的加速率高于CPU。

对写吞吐量的影响

![图5.不同data size下的写吞吐量](https://img-blog.csdnimg.cn/f8a8ae5309ad4945bea69e6a4e5bd340.jpeg#pic_center)

如图5可见，data size的增加导致CPU和FPGA两者的写吞吐量都下降，但FPGA下降的较为平缓，同时FPGA增加了系统的写吞吐量。

在多路输入的情况下

![图6.不同value length下的合并速度比较](https://img-blog.csdnimg.cn/a7a67f51f5b44965be61248493d5693a.jpeg#pic_center)

![图7.不同value length下的加速率比较](https://img-blog.csdnimg.cn/b255752fcc42439b81ef8125a9bc6066.jpeg#pic_center)

把输入从两路扩展为九路，可以从图6和图7中看出，在九路情况下FPGA的合并速度低于两路的情况，但差距随着value length的增加逐渐缩小，同时它的总体加速率实际上更高了。

Data Size大小变化产生的影响

![图8.不同data size情况下的写吞吐量](https://img-blog.csdnimg.cn/350f36d6f787497fb4375d9bfb2bf789.jpeg#pic_center)

如图8所示，无论是否使用了FPGA，随着data size大小的增加，写性能都显著下降，同时FPGA的加速率也减少，最后稳定在2.5左右。尽管数据大小的增长可能使得传输时间变得不可忽略，但实际上数据传输时间在整个系统执行时间内所占比例依然不足1%。

LevelDB的设置对系统的影响

![图9.Key length、Value length、Data block size以及Leveling ratio对系统的影响](https://img-blog.csdnimg.cn/2b3f22fb333c4c628d4a598e1b0a80c3.jpeg#pic_center)

如图9，(a)表明Key length对系统的影响是线性变化，随着Key length增大，加速率会线性下降。
(b)表明加速率会随着Value length的增加而增加。
(c)表明Data block size对于系统几乎没有影响。
(d)表明Leveling ratio的增加会使得加速率下降。其中Leveling ratio为Size(Level i+1)/Size(Level i)，这个数值增加表明合并被触发的越不频繁，从而FPGA加速的机会被减少。

文中还使用了YCSB进行测试，发现即使是在只读工作环境下，吞吐量依然没有减少，因为对系统的修改并没有涉及到存储结构。

**结论**

文中提出了一个使用FPGA对LSM-tree based key-value stores进行加速的方案。而最终实验结果也表明，该硬件可以达到92倍的加速率，获得6.4倍的吞吐量增加。