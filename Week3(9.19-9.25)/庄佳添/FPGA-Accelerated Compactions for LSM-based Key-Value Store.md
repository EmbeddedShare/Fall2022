## FPGA-Accelerated Compactions for LSM-based Key-Value Store 

### Background

LSM-tree依赖compaction操作实现高效写操作。然而，作者发现采用LSM-tree的KV store存在着处理查询和事务的CPU资源 和 compaction操作资源之间的竞争问题，导致compaction慢，尤其是在WPI(写和点查询密集)工作负载下。Figure 1展示对一个WPI workload(75%读25写)不同compaction线程的吞吐量情况，当线程大于32时性能反而下降。

![image-20220921145709213](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209211457247.png)

### Motivation

现有的LSM-tree KV store还存在如下问题。1. L0的数据块从内存直接写入并没有经过merge操作，有重复的key值，较慢的compacton操作使得L0数据块的积累，1次点查询需要在L0搜索多个块，造成额外的开销。且刚输入L0的数据被再次访问的可能性较高。2.compaction 瓶颈的转移。图3显示随着值大小的降低，IO所占的比重增高，计算比重降低，这表明随着值体积的增大，compaction操作瓶颈从CPU上转移到IO上，当合并小体积的KVs时性能受限于计算资源，合并大体积KVs时受限于IO资源。

![image-20220921151407350](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209211514396.png)

本文将compaction操作下发到FPGA上进行加速，更快的实现compaction操作并释放CPU性能。FPGA具有如下优势1. 减少cpu开销提高cpu性能 2.fpga适合于流水计算的特点 3. fgpa的高能源效率 4.经济效益。FPGA与KV Store结合由2种方式，FPGA的板上RAM较小时通常被放置在CPU和硬盘之间作为**数据过滤器**；FPGA的板上RAM较大时可以作为**协同处理器**处理较大量的数据，本文使用该方式。



### Design and Implementation

1. Figure 4展示系统整体设计。Task Queue缓存新触发的compaction任务，Result Queue缓存compact过后的KV记录。软件方面引入了Driver下发compacton任务和管理数据在主机和FPGA之间的转移。FPGA上部署多个compaction单元负责合并KV记录。

   ![image-20220919210508141](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209192105188.png)

2. Driver的第1个功能是管理compaction任务。系统维护3类线程。

   ![image-20220919213218615](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209192132646.png)

   Builder线程将将要merge的extents分割成大小相似的组，每一组分别形成一个compaction task，被追加到task queue中。当1个task成功完成后builder线程将已经compacted的数据块持久化，否则启用cpu compaction线程重试该task。每个compaction task应该包含的元数据如Figure 6所示。

   ![image-20220919213152271](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209192131309.png)

   Dispatcher线程通过轮询分配的方式将tasj分配给FPGA上的compaction单元(CU)，由于每个task的大小相近，轮询分配的方式可以让cu达到负载分配平衡，最后dispatcher线程通知driver线程将task任务的数据写到FPGA的内存上。

   Driver线程将与一个task相关联的数据写入FPGA的内存中并通知相应的CU单元开始工作。当compaction任务结束，compaction线程被中断，将数据从FPGA内存写回主机内存，并将task追加到result queue。

   同时，如Figure 7所示，作者为driver和fpga设计了专门的数据和指令路径、中断机制、FPGA内存管理单 元。数据通路使用DMA方式，使CPU不参与到数据的转移中来。当1个compaction任务完成后，中断机制发挥作用使得相应的线程将结果写回主机内存。内存管理单元MMU为FPGA分配内存单元。

   ![image-20220919214830379](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209192148425.png)

3. compaction单元(CU) 结构如Figure 8所示。

   <img src="https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209211107272.png" alt="image-20220921110716140" style="zoom:50%;" />

   考虑到KV store中键值可能有前缀压缩，作者引入Decoder对输入的kv记录解码。 采用leveling compaction策略，一次最多只有来自相邻层2路数据输入；采用tiering compaction策略，一次有2-4路数据输入。因此，按1次最多以4路数据输入考虑，若每个CU中只设置2个Decoder(2Decodere,1Merger)，则需要3次的compaction操作；若每个CU中设置4个Decoder(4Decodeer,1Merger)，则可以在多用40%硬件资源的情况下减少多余的compaction任务。

   KV Ring Buffer用来缓存解码后的KV记录，作者观察到大多数KV记录不大于6KB，因此设置每个每个KV ring buffer包含32个8KB大小的槽，额外的2KB用于存储元数据。每个KV Ring Buffer具有3种状态：满、空、半满。一开始KV Ring Buffer为空状态，此时Decoder开始解码并将解码后的KV记录写入KV Ring Buffer；当达到半满状态时，Merger开始读取KV Ring Buffer并进行合并，此时Decoder持续往KV Ring Buffer中剩余的槽中写入数据直到达到满状态。理想状态下，匹配merger和decoder的速度使得merger完成一次合并操作的时间与Decoder写入一半KV Ring Buffer大小数据的时间相等，此时decoder和merger形成流水工作；二者速度不匹配时Controller负责暂停decoder工作。

   当多路的key被写道Key Buffer后，Merger开始进行比较。当某一个key确认为输出后，KV transfer复杂将对应的KV记录从KV Ring Buffer读入KV Output Buffer，此时Controller将该KV Ring Buffer上的读指针向后移动。最后Encoder将KV output Buffer中的KV 记录打包成数据块并写入到CU 内存。

4. 因为流水工作的设计，为避免过度提供和资源浪费，作者提出一套CU分析模型。

   1.  Equation1所示，CU的吞吐量由最小吞吐量的环节决定。唯一的特例是，当KV Transfer将已合并的记录写入KV Output Buffer时，他的吞吐量应该与Merger合并计算。

       ![image-20220920150547935](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201505039.png)

   2.  Equation 2、3、4、5分别计算decoder、merger、kv_transfer、encoder的吞吐量，其中b表示启动该阶段工作的基本时间周期。**(*A放大因子怎么理解？？)***

      ![image-20220920152751620](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201527659.png)

   3. Equation 6计算FPGA和主机间数据转移的吞吐量。N是处理的KV记录数。***(公式、u如何理解？？)***

      ![image-20220920153637101](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201536130.png)



### Evaluation

#### 评估FPGA-based compaction

1. 为评估CU的性能，作者分别使用CPU和FPGA对400万条KV记录进行compaction。实验结果显示FPAG compaction 的吞吐量比 CPU compaction的吞吐量高203%到507%。对于长度较小的KV 记录，将任务下发到FPGA的开销占总开销的比率较高；相反，长KV记录下发到FPGA的开销占比较小，因此有更好的加速效果。![image-20220920161219644](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201612686.png)

2. 作者使用100%的选择率(所有的额KV记录都被解码、比较、编码和持久化)验证CU分析模型。当value值小于25比特时模型评估结果的错误误差不超过5%，随着value值增大误差增大，1024字节的value对应的误差为13%。这种误差由流水线推迟和总线竞争导致。总体吞吐量受限于Merge和KV Transfer的串行执行。

   ![image-20220920163541322](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201635369.png)

3.  Table2展示1个CU对资源的利用情况及其对FPGA全部资源的利用率。在实验中最多可以在FPGA中防止8个CU(Utilization * 8),外加FPGA上约15%的开销，作者的实现对FPGA资源的利用率约为50%。

   ![image-20220920164728087](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201647125.png)

   #### 评估FPGA-offloading of compactions对KV Store的影响

   1.  Figure 12展示不同compaction线程下和PPGA加速下的整体的吞吐率和L0的数据块大小。线程数小于24线程时，这个阶段的compaction线程数不足以应对KV store写入的总体数据量，因此使用更多的compaction带来显著的性能提升。线程数在24到32时，整体吞吐量几乎不变，线程数大于32时，整体性能下降。原因如下：1. 线程数的增加导致资源竞争(Figure看出此时的CPU利用率100%)，且此时compaction受限于CPU(Figure 3表示 value大小为32比特时受限于cpu)。2.更多的线程使得磁盘IO饱和，导致事务处理和compaction的IO竞争(Figure 1)。Figure12表示采用FPGA-offload compaction的吞吐量比用CPU进行compaction的吞吐量高23%，得益于FPGA加速和减少cpu资源竞争。

      ![image-20220920165643869](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201656905.png)

      Table3总结了采用FPGA offload compaction和CPU-based compaction(32线程)的性能表现和能源消耗情况。![image-20220920193113460](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201931501.png)

   2. Figure13探究不同读写比率的影响。FPGA offload compaction比cpu compaction降低了10%的CPU消耗，并且有更高的吞吐量。

      ![image-20220920194027737](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201940794.png) 

   3. Figure14展示FPGA加速compaction和CPU compaction两种方式的DBbench的测试结果。FR随机写，SRWR搜索某条随机记录并同时有一个线程写数据；RWR多条线程随机读一条线程随机写；UR随机更新；RRWR所有线程90%时间随机读10%时间随机写。测试结果表明FPGA offload compaction在IO数基本不变的前提下提高了系统吞吐量并降低了CPU使用率。其中FR的性能提升最明显，因为FR的大量写操作使得compaction操作数量的增加；RWR受限于IO，FPAG offload通过释放CPU处理compaction IO部分的性能，使得整体吞吐量提高18%。![image-20220920194630760](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201946809.png)

      Figure 15展示YCSB的测试结果。workload a50%读50%更新；b 95%读5%更新；c 只读；d 95%读5%插入；e 95%扫描5%插入；f 50%对最新记录先读后写 50%随机读。这写workload都已读操作为主，因此compaction次数不多，因此FPGA offload吸能提升并不明显。![image-20220920194650704](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202209201946757.png)

#### Conclusion

本文提出将compaction操作下发到FPGA上执行，并与X-Engine结合进行试验。与使用CPU进行compaction的系统相比较，本文提出的方式在系统吞吐量和能源效率上分别提高了23%和31.7%。