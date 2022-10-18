# **MatrixKV: Reducing Write Stalls and Write Amplification in LSM-tree Based KV Stores with Matrix Container in NVM**
Author :Ting Yao, Yiwen Zhang, and Jiguang Wan, Huazhong University of Science and Technology; Qiu Cui and Liu Tang, PingCAP; Hong Jiang, UT Arlington; Changsheng Xie, Huazhong University of Science and Technology; Xubin He, Temple University

当前的LSM-tree based key-value store依然有着次优的性能和不可预测的表现，这主要是受到了Write stall和Write amplification两个因素的影响。已经有不少文章尝试对这两个问题提出解决办法，但尚未有同时考虑两者的文章，本文章利用了NVM，并提出MatrixKV来同时解决这两个问题。

## **Abstract**

LSM-tree based key-value store的性能受到了Write stall和Write amplification的影响，前者主要来自于LSM-tree的前两层，即第0层和第1层之间的合并。而后者会随着LSM-tree的深度的增加而变大。Write stall会阻塞数据的输入，导致吞吐量降低，性能不稳定并产生长尾延迟，而WA会降低系统性能，影响存储设备的寿命。
为了解决这两个问题，文中提出了一种适合多层DRAM-NVM-SSD存储系统的LSM-tree based KV store，MatrixKV，并且应用了NVM。这一系统设计的主要思路是通过把前两层的合并变为细粒度，而且放在更加实惠的NVM上执行，同时将传统的LSM-tree压扁，通过增加每层的大小来保持每层大小比例不变的同时减少层数，以此减少两个问题带来的影响。
为了实现上述设计，文中提出了四个新技术：martix container,column compaction,增加每层的宽度以及cross-row hint search，前两个是为了适配在NVM中处理前两层的合并，第三个是为了解决WA问题，最后一项技术是为了避免在应用NVM后使得读性能下降太多而做的优化。

## **Background**
**Non-volatile Memory**

NVM，非易失性内存，这种设备在不丢失性能的前提下具有Byte-addressable，persistent的性能。它可以提供接近DRAM的性能，像disk一样的持久性，同时相比于DRAM有更高的容量和更低的价格。
NVM可以通过PCIe接口作块设备，也可以通过内存总线作持久性的主存设备。但当其作为主存时，相比于DRAM，NVM依然有5倍的带宽差距和3倍的延迟差距。

**Log-structured Merge Trees**

LSM-tree的设计理念是将随机写转化为顺序写，从而充分利用硬盘的顺序写性能。

LSM-tree中包含内存部分和硬盘部分，内存部分会在大小达到一定程度后flush到硬盘中，成为硬盘部分。硬盘中的KV存储是分层有序存储，当前层次大小到达一定程度后，需要与下一层有key重叠的部分合并然后移动到下一层。随着数据的增加，LSM-tree的深度也会增加。

## **Motivation**
 为了探索影响LSM-tree based KV store性能的主要因素，文中对SSD-based RocksDB进行了一个实验，将80GB的数据集随机写入RocksDB，实验结果如图1
 
![图1.RocksDB的随机写性能和前两层的合并](https://img-blog.csdnimg.cn/cfdcab8e2b9b4b6785067f7a4ea688e2.jpeg#pic_center)

图中的吞吐量低谷表示Write stall，这表明Write stall使得系统性能波动，导致不可估计性和不稳定性。而系统平均性能也随着时间的增加而下降，因为数据越多，LSM-tree层数越多，WA越大。因此两个主要因素是Write stall和WA。

**Write stalls**

Write stalls一共有三种情况：
1、Insert stalls：Memory table满，insert操作会被阻塞。
2、Flush stalls：如果L0层已满，则flush操作会被阻塞。
3、Compaction stalls：当有太多合并发生时，前台的操作会被阻塞。
对所有这些阻塞分别记录后，文中发现前两层的合并时期大约与观察到的Write stalls发生时间相匹配。

**Write Amplification**

WA指写入存储设备的数据量与用户写入的数据量的比值。LSM-tree based KV由于每层之间的合并操作，WA的值与每层之间的放大系数（AF）有关。每两层之间的WA为AF，因此总的WA为（层数-1）*AF，层数越多，WA越大。

**NoveLSM**

NoveLSM也利用了NVM来为DRAM-NVM-SSD存储系统提高性能，因此选择NoveLSM来做评价的比较对象之一。

## **MatrixKV Design**

Matrix KV的设计如图2所示

![图2.MatrixKV的架构总览](https://img-blog.csdnimg.cn/97065d02bae74385b2b3930dd5a28620.jpeg#pic_center)

该架构适用于多层DRAM-NVM-SSD系统。其大致工作流程为：
1、DRAM分批写MemTables。
2、MemTables被flush到L0中，L0由NVM里的matrix container存储和管理。
3、L0中与SSDs中的L1通过column compactions来合并。
4、SSDs存储被压扁的LSM-tree的剩下的层。

**Matrix Container**

Matrix Container的设计如图3所示

![图3.Matrix container的结构](https://img-blog.csdnimg.cn/6be9c6eea15f4b458432803c7f053dd1.jpeg#pic_center)

Matrix container是为了能将L0和L1的合并变为细粒度，并卸载到高速的NVM上进行处理。它由Receiver和Compactor两部分以及一个Space management组成。
Receiver用于接收和维护从DRAM中flush来的MemTables，每个MemTable会被序列化为receiver的一行并且被组织为RowTable，RowTable的结构如图4所示，包含数据和元数据，元数据中每个元素包含一个用于cross-row hint search的forward pointer。

![图4.RowTable的结构](https://img-blog.csdnimg.cn/ed04ed798986444c842fc7ef88a08c05.jpeg#pic_center)


RowTable会被添加到matrix container的后面。当receiver的大小达到一定程度，并且此时compactor为空时，这个receiver就会停止接受DRAM的数据，并且变为一个compactor，这一变化过程没有发生实际的数据传输。
Compactor用于以细粒度方式选择和合并L0和L1的数据，从而缓解了合并时带来的Write stalls。
当compactor对一列完成合并后，NVM的空间会被立即释放，并由space management按页为单位进行管理。

**Column Compaction**

Column compaction是L0和L1之间一种细粒度的合并方式，每次只合并一个逻辑列，这一个逻辑列只包含一个特定的键值范围内的元素。其工作流程为：
1、MartixKV将L1的键值空间分为多个不同区域。
2、Column compaction从第一个键值区域开始，作为合并区域。
3、每次合并有多个行并行处理，假设有N行，一共有k和线程，则每个线程负责N/k个行，其中k=8。
4、如果当前键值区域内的数据量未达到一个给定的下限，则继续选择下一个键值区域内的数据，直到数据到达下限和上限之间为止。
5、此时一个逻辑列已经形成，合并开始，直到整个键值区域都被排序完成。
示例如图5所示

![图5.Column compaction的一个例子](https://img-blog.csdnimg.cn/169343f0f91a429aa8f8ff8eb737f5a4.jpeg#pic_center)

实例中两条红色虚线之间的数据为一个逻辑列，和对应的L1中的SST文件相合并。

**Reducing LSM-tree Depth**

文中减少LSM-tree深度的方式即增加每层的大小。
这种方式会使得L0增大，L0与L1合并时会加剧Write stalls，同时也导致读数据时搜索变慢。
对于第一种问题，Column compaction可以解决，对于第二种问题，利用了Cross-row Hint Search。

**Cross-row Hint Search**

Cross-row Hint Search即在RowTable的元数据中给每个元素添加一个forward pointer，这一指针指向其下一行RowTable中第一个不小于该元素的元数据为止。
具体查找过程如图6所示

![图6.Cross-row hint search示例](https://img-blog.csdnimg.cn/f0a2d2dd1a2f40dd8d49b66801a41087.jpeg#pic_center)

图中展示了利用二分查找寻找12的过程，蓝线即表示forward pointer，在第一行无法找到12的时候，就从包含了12的键值范围的相应的forward pointer往下一行寻找。

## **Evaluation**

总体性能如图7所示

![图7.不同value size下的性能](https://img-blog.csdnimg.cn/46a5925f52d44177876d6f7458213771.jpeg#pic_center)

根据图7，可以看出MartixKV在随机写上性能很好，顺序写和读性能也没有下降。

![图8.随时间变化的吞吐量波动](https://img-blog.csdnimg.cn/165e93fffe0f4e83acc8c48f46cadff5.jpeg#pic_center)

根据图8中的低谷可以看出，MatrixKV很好的缓解了Write stalls。

![图9.NVM-based KV store的尾延迟](https://img-blog.csdnimg.cn/016d84b36b2c440793a0d5cf3fe203d9.jpeg#pic_center)

根据图9可以看出，即使是在同样使用NVM的情况下，MartixKV的尾延迟依然低于大部分其他设计。

![图10.80GB数据随机写的写放大](https://img-blog.csdnimg.cn/1c488ad0d3e34eb49c6dbbaf3c58f6da.jpeg#pic_center)

根据图10可以看出，MatrixKV也很好的降低了WA的值。

## **Conclusion**

文中提出了一种低延迟的稳定LSM-based KV store，MatrixKV。这种设计配合多层DRAM-NVM-SSD存储系统，缓解了Write stalls和Write amplification问题，极大提高了系统的整体性能。