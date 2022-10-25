# **RM-SSD: In-Storage Computing for Large-Scale Recommendation Inference**

Author：Xuan Sun,Hu Wan,Qiao Li,Chia-Lin Yang,Tei-Wei Kuo and Chun Jason Xue

Department of Computer Science, City University of Hong Kong,
Department of Computer Science and Information Engineering, National Taiwan University,
NTU High Performance and Scientific Computing Center, National Taiwan University

为了满足推荐系统的严格服务要求，系统的整套嵌入组件都被装载到内存中。但由于产品级的推荐系统的模型和数据集的规模逐渐增大，嵌入组件的大小逐渐接近内存的容量上限，这种有限的空间限制了算法，从而限制了推荐系统的发展。之前的研究致力于embedding-dominated的模型，而这篇文章提出把整个推荐系统装载到带有存储设备内计算能力的SSD中，利用FPGA同时加速embedding-dominated和MLP-dominated的模型。最终相比于基础SSD和先进的方式分别实现了20-100倍和1.5-15倍的吞吐量提升。

## **INTRODUCTION**

推荐系统是基于用户的兴趣和偏好提供私人推荐的系统，推荐系统基于深度神经网络。推荐系统除了基本的MLP层以外，还具有一个数据密集的嵌入层，用于处理输入的稀疏分类特征。在嵌入层，稀疏的特征会被映射到embedding vectors里，这些向量存储在多个embedding tables中。处理时，系统会根据查找索引从embedding table中提取向量，不同表中的向量被聚合后，embedding lookup的结果会被发送到MLP层作进一步处理。

Embedding tables的快速增长导致模型的规模变大，在DRAM中存储embeddings变得不现实，为了解决这个问题，可以将embedding tables存入SSD，但这会导致系统的延迟变大，吞吐量降低。而基于embedding层和MLP层的执行时间，推荐系统模型可以分为embedding-dominated和MLP-dominated的模型。其中前者的性能问题主要由读增大和不规律的访问模式导致。

这篇文章提出了RM-SSD，这是一个带有FPGA-based in-storage计算引擎的SSD。文中将整个推荐系统装载到了RM-SSD上，这一解决办法可以同时提高两种模型的性能，并且将embedding tables存储到SSD上。对于embedding层的延迟问题，文中提出了Embedding Lookup Engine，同时提出了MLP Acceleration Engine来加速MLP层。除此之外，文中还实现了一个host和SSD之间的推荐系统语义感知接口来优化延迟和吞吐量。评估后，发现RM-SSD相比于基础SSD和先进的SSD分别获得20-100倍和1.5-15倍的吞吐量提升。

## **BACKGROUND**

**Recommendation Systems and Models**

![图1.推荐模型的结构](https://img-blog.csdnimg.cn/85ba0f2df96f478fa6d001f1c06a3381.jpeg#pic_center)


先进的推荐模型利用深度学习算法，如图1，模型的输入是一系列密集和稀疏的特征。密集的特征会被底部MLP层处理，稀疏的特征会通过embedding table lookup转化为密集形式，转化后的结果会和密集特征通过特征交互而产生联系，之后的输出会被顶部MLP层处理，最终产生一个代表特定事件的可能性的值。可以看出，推荐系统需要很大的内存空间，并且数据密集的sparse embedding操作通常是瓶颈。

**Opportunities for In-Storage Computing**

SSD之所以适合装载部分计算工作，是因为不平衡的带宽和冗余数据的移动。多级并行的flash阵列中，内部和外部带宽通常无法匹配，若将计算直接放入SSD中，可以充分利用不同层级之间的并行性。而在数据处理过程中，如果直接在SSD内部进行计算，就可以减少冗余数据传输的时间。

## **MOTIVATION**

**Challenge of Large-Scale Recommendation Systems**

规模增大的推荐系统模型无法继续装入DRAM，而单纯装入SSD会导致性能下降，因此主要面对的挑战就是使装入SSD的推荐系统获得一个可以接受的延迟和吞吐量。

**Performance of Naive SSD Deployment**

![图2.不同SSD部署的性能](https://img-blog.csdnimg.cn/d533f3c1b2d944aab342e315f0d3b8b1.jpeg#pic_center)


文中所做的性能测试如图2，相比于DRAM-only的版本，SSD的性能表现下降很明显，在内存大小有限时更为突出。MLP-dominated的模型的性能下降更小，因为它的MLP操作占据了超过一半的时间。SSD版本下大部分时间花费在了SSD的访问和文件系统I/O栈上，因此文中提出in-storage计算来缓解这个问题。

![图3.不同设置下的读增长](https://img-blog.csdnimg.cn/452f46b99f5a43db84785eb3c964f450.jpeg#pic_center)


如图3，以SSD为基础的推荐系统的写放大提高了几十倍，因为当DRAM减小后，缓存空
间也减小，使得系统访问SSD更加频繁，同时因为传统的SSD只支持页粒度访问，不支持
向量级别的访问，因此冗余数据较多。

![图4.Embedding Vector访问模式](https://img-blog.csdnimg.cn/a5cb8e45c7a9471681f929be0950f76f.jpeg#pic_center)


如图4，embedding的访问模式不规律，只有很小一部分数据有较高的访问频率，这不具有
空间相关性的模式，即使增大缓存容量，带来的提升也很小。

MLP模型由于以下几个原因适合装载于带有FPGA的SSD上
1、对于MLP层，FPGA比CPU运行更快
2、系统性能吞吐量可以用流水线提升
除此之外，将流水线的所有阶段放在存储设备中比host和SSD共同设计的流水线机制更加有效，因为整个系统的同步开销很大，同时额外数据的复制和格式调整也会带来很大延迟，而且对于在SSD侧的embedding和在host侧的MLP分别做优化会被两层之间的内部关联阻碍。

## **RM-SSD DESIGN**

**System Overview**

![图5.RM-SSD的结构](https://img-blog.csdnimg.cn/ebe290804b1c4122adc902c24dfab686.jpeg#pic_center)


如图5，RM-SSD由常规组件、Embedding Lookup Engine和MLP Acceleration Engine组成。

常规组件包括NVMe Controller，Flash Translation Layer和Flash Memory Controllers。它们分别用于处理host和SSD之间的通信，逻辑块地址和物理块地址之间的转换以及相应闪存通道和外部控制器之间的数据传输。

MMIO Manager用于解决I/O栈问题，支持host端的内存接口和DMA模式下的数据传送。RM-SSD可以接受block I/O request和reccommendation inference，前者被交给NVMe Controller处理，后者由MMIO Manager处理。

**Embedding Lookup Engine**

![图6.EV Translator的设计](https://img-blog.csdnimg.cn/29aa9b4945464dffa091592076b3f8cb.jpeg#pic_center)


Embedding Lookup Engine由EV Translator，EV-FMC和EV Sum组成，如图6。这一组件可以解决读放大问题。

当一批embedding vector lookup到达时，host端告知EV Translator查询的数量并以DMA模式将索引传送到Index Buffer中，当查询的数量传送结束后，EV Translator会扫描每个表的元数据，随后Read EV Req会每次从Index Buffer中读取一个索引，通过检查index range，可以得到当前索引的extent ID，根据相应的extent ID，可以确定一个Start LBA，而之后根据索引的偏移量和向量维度，可以获得该embedding vector的最终LBA，从而实现向量级别的读。

生成的LBA会被FTL转为PBA，之后物理块读请求会被分配给相应的EV-FMC，MUX是用于区分传统的block I/O request和reccommendation inference。随后从flash channel中返回的数据会被DEMUX根据Path Buffer中的flag作出区分，属于block I/O request的数据会在收集完成一整页后交予NVMe Controller处理，而embedding vector的读操作会被EV Sum处理。

**MLP Acceleration Engine**

在MLP层和embedding之间由密切的内部关联，这种影响可以由Intra-Layer Decomposition消除，同时，推荐模型由多个FC层组成，这些模型可以同时全部装入FPGA，因此延迟和吞吐量可以被进一步通过Inter-Layer Composition优化，而且不平衡的结构提供了平衡各个FC层之间的时间开销的机会，除此之外，embedding层也可以通过利用Kernel Search Algorithm最大化数据处理吞吐量来于MLP层取得平衡。

相比于使用脉动阵列来实现Matrix Multiplication，我们使用adder tree来实现kernel block MM中的sum操作。

![图7.Decomposition优化](https://img-blog.csdnimg.cn/4a536fa1da604555b705fe3016e6357b.jpeg#pic_center)


底部的MLP和embedding层由于feature interaction操作产生内部联系，可以将输入分为bottom MLP输入和embedding输入，使得两者并行处理，直到相应的结果求和完毕，这种分解操作使得第一个FC层的性能不会被bottom MLP层和embedding层影响。

![图8.Composition优化](https://img-blog.csdnimg.cn/b4619725efd34b45b54ccfa6588fdfbb.jpeg#pic_center)


若令放入FPGA的所有FC层都使用同一种扫描模式，则会产生流水线阻塞，因此这篇文章提出使用Inter-layer composition，当Li被按列扫描时，按行扫描Li+1，如图8，从而减少MLP消耗的时间。

Matrix Multiplication对于RM-SSD的资源利用至关重要，因此提出一个kernel search algorithm来决定每个FC层的内核大小，从而确保最低的资源消耗和优化的吞吐量。该算法需要遵守四个规则：
1、优先将所有任务分担给BRAM，如果

![图9](https://img-blog.csdnimg.cn/2ea7630525484affa44b6b16ac27fc56.jpeg#pic_center)


，则使用DRAM。
2、若DRAM被安装在一个特定的FC层，那么这层的瓶颈就是DRAM的读带宽。系统中会使用两重缓存来避免读阻塞，当buffer A为了IIB而被保留时，buffer B会被用于读取大小为IID的数据，为了充分利用带宽，IID的值必须>=IIB。
3、默认batch size=1，但若

![图10](https://img-blog.csdnimg.cn/f34d29100d924a7eb72d6fc9a747bd4d.jpeg#pic_center)

![图11](https://img-blog.csdnimg.cn/82e9b33d25da4d1b9d0783663363934f.jpeg#pic_center)

，则需要增加batch size的值直到两个大于号都变为小于等于为止。
4、为了避免流水线气泡，必须满足

![图12](https://img-blog.csdnimg.cn/64cfc6205b2443e28e4b6305ed046a22.jpeg#pic_center)


## **EVALUATION**

EMB-VectorSum代表只包含Embedding Lookup Engine的RM-SSD。

**SLS operator performance**

![图13.SLS操作的性能](https://img-blog.csdnimg.cn/6e2cad55b53744d4967776cebb904fe9.jpeg#pic_center)


如图13(a)所示，EMB-VectorSum的性能是基础的SSD-S的16倍，如图13(b)所示，EMB-VectorSum的执行时间会随着查找规模的增大线性增长。

**End-to-End performance**

![图14.End-to-end的性能](https://img-blog.csdnimg.cn/bf87cc380c2f47609d64abf67d4f12db.jpeg#pic_center)


如图14所示，EMB-VectorSum相比于SSD-S，实现了17倍的加速。

**Evaluation of Full RM-SSD**

![图15.不同实现的吞吐量](https://img-blog.csdnimg.cn/26ec14e50f404586a62a9cfaf446405e.jpeg#pic_center)


如图15所示，RM-SSD相比于基础SSD-S，实现了36倍的吞吐量提升

![图16.不同实现的延迟](https://img-blog.csdnimg.cn/8f0f6c3f3f1448e7892339e1fba7120f.jpeg#pic_center)


如图16所示，RM-SSD相比于基础SSD-S，延迟减少了97%

## **CONCLUSION**

这篇文章提出了RM-SSD，一种使用低端FPGA在内存有限的系统下加速大规模推荐判断的in-storage计算原型。这篇文章解决了embedding-dominated和MLP-dominated两种推荐模型的延迟和吞吐量问题，降低了embedding lookup的读放大，加速了MLP层，相比于基础SSD实现了20-100倍的吞吐量提升，相比于最新研究实现了1.5-15倍的提升。
