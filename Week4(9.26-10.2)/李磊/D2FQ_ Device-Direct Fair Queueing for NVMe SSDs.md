## D2FQ: Device-Direct Fair Queueing for NVMe SSDs
>作者：
Jiwon Woo, Minwoo Ahn, Gyusun Lee, Jinkyu Jeong
Sungkyunkwan University
FAST2021: File and Storage Technologies
[原文链接](https://www.usenix.org/conference/fast21/presentation/woo)

总结：论文基于现代SSD 带宽和容量的巨增以及它的并行处理能力，典型I/O调度器在操作系统的I/O块层中进行I/O请求的提交、仲裁、分配，而这三步在CPU利用率、可伸缩性和块I/O性能方面会产生很高的开销。因此提出了将I/O调度功能卸载到设备上，并参考虚拟时间机制和NVMe加权轮询(WRR)仲裁的特性，实现了公平共享SSD性能和节省工作量。最后对合成负载和现实工作负载做了实验，评估结果表明，D2FQ提供了公平性、低CPU利用率和高块I/O性能。

### 1.背景和动机
- **Fair Queueing for SSDs**
现代高性能ssd能够容纳来自多个租户的并行I/O请求，ssd的带宽和容量的巨大增加使其能够在单个存储设备中为来自多个独立工作负载(或租户)的I/O请求提供服务。因此，SSD性能的公平共享对于在多个应用程序或租户之间提供性能隔离非常重要。
在许多比例共享I/O调度器中，基于虚拟时间的公平队列由于其节省工作的特性而成为SSD的一个有吸引力的解决方案。它们可以最大化SSD吞吐量，而每个租户实现的带宽与租户的权重成正比。

- **I/O Scheduling Overheads in Software**
操作系统I/O堆栈中的块层是块I/O调度的核心。通用块层提供I/O请求的合并、重新排序、暂存和记帐。典型的块I/O调度器在I/O处理期间需要执行三个步骤(提交-仲裁-分派)，如图1（a）。然而，对于高性能SSD，块层会增加CPU周期和I/O延迟方面的开销。因此，许多研究建议绕过块I/O层以实现低I/O延迟。
![在这里插入图片描述](https://img-blog.csdnimg.cn/77f5bdf31c874506ac6d78fc50acdc86.png)

下图三种调度方法比较了延迟和CPU利用率。without I/O scheduling (None), MQFQ, and the light-weight block layer (Bypass，绕过块层并将I/O请求直接提交到设备的命令队列。)。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b3cdc3daae454e069e91a6c20817d6ed.png)
Bypass 显示最低的I/O延迟、最高的I/O带宽和最低的CPU利用率。由于其块I/O服务的CPU开销最低，它比其他方案更能延迟饱和点。
将I/O调度功能卸载到设备上是减少CPU开销同时保持公平性的一种有吸引力的方法。由于许多网络接口卡具有设备端I/O调度特性，这种调度卸载已经广泛应用于网络包调度领域。网络接口卡早于ssd经历了微秒级I/O时延的时代。许多方法都提出了将数据包调度转移到网卡上，并成功地降低了CPU利用率。所以见上图1（b）中的调度流程，实现提交即分配。
- **Weighted round-robin in NVMe Protocol**
NVMe协议有一个称为加权轮询(WRR)队列仲裁的块I/O调度特性，协议默认的I/O命令调度策略为round-robin;因此在命令队列中逐一处理I/O命令，如图3a所示。如果启用WRR功能，SSD固件将以加权的roundrobin方式获取I/O命令，如图3b所示。NVMe WRR特性可以很容易地在SSD内部实现，因为它很简单。
![在这里插入图片描述](https://img-blog.csdnimg.cn/aaf4375b572846248087c76f462ff566.png)
详细可参考[关于NVMe的命令仲裁机制](https://mp.weixin.qq.com/s?__biz=MzIwNTUxNDgwNg==&mid=2247484375&idx=1&sn=d7854b5dddd0407a24753b9da176e40a&chksm=972ef28ea0597b98c60ddf0e2f62ff80be42a60e5c2a0ae9429e51644bd3b74152e3dd4ec684&scene=21)

### 2. 挑战
基本的NVMe WRR太简单了，无法正确地调度来自多个租户的I/O请求，这些租户具有不同的I/O特征，比如不同的线程数量、不同的I/O请求速率和不同的请求大小。
首先，NVMe WRR只有三个优先级类(即low、medium和high)，而租户的数量可以更高。其次，它的队列仲裁不考虑I/O大小。最后，任何两个队列之间的权重比都不能直接匹配这两个队列服务的I/O命令的比例。这是因为实际处理的I/ o数量可能根据命令队列的利用率而变化。
### 3. 设计
#### （1）technique
论文提出了一种公平排队调度程序D2FQ，将I/O调度功能卸载到SSD设备上，基于SSD NVMe加权轮询(WRR)仲裁的特性和参考虚拟时间机制，D2FQ将WRR中的三类命令队列抽象为三个具有不同I/O处理速度的队列，通过动态调整三种不同处理速度队列的处理数，缩小虚拟时间差距，满足公平性。
#### （2）solution
D2FQ的框架如下图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/36a981b8e9b14faa819e3cd26606171d.png)
注意：方案假设所有core都有自己的队列集(每个优先级类有三个队列)。
由于D2FQ将I/O提交和调度步骤统一为一个步骤，因此它可以与无I/O调度功能的低延迟I/O堆栈集成。
伪代码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/caaccca7dc774d93a0121ec2cbab2767.png)
参数和变量的控制：
- **1. gvt(global virtual time)**
![在这里插入图片描述](https://img-blog.csdnimg.cn/59283b7a44864567b43e62ccfbad8abf.png)
当一个流f被一个长度为l的I/O请求服务时，该流的虚拟时间增加l/wf，其中wf是流的权值。
理论上，gvt = 所有活动流（有挂起的I/O请求的流）之间的最小的虚拟时间。
跟踪gvt等于跟踪一组值中的最小值，其中每个值是同时变化。但是在D2FQ中，采用了另一种粗略地跟踪gvt的方法。
论文认为不总是需要在流的虚拟时间中检索真实的最小值。跟踪gvt的目标不是最小化任何流之间的虚拟时间间隔。它是使任何流的虚拟时间进程速度以相似的速度进行。
![在这里插入图片描述](https://img-blog.csdnimg.cn/7423d3cf0eb2466987938693200566a3.png)

论文提出的方案维护一个gvt持有者，即拥有gvt的流，并且只允许gvt持有者能够增加gvt（图9中的第9-13行）。其他流也可以更新gvt，但仅当其虚拟时间小于gvt时（第11行）。当gvt持有者增加gvt并且它现在超过其他流的虚拟时间时，可能会出现一点不准确，因此违反gvt不是最小值。然而，由于gvt更新操作的简化，这一点不准确；否则，每次gvt更新都需要检查所有流的虚拟时间。
函数update_gvt（）在I/O完成时调用。当gvt持有者变为非活动状态时，它调用release_gvt（），任何流都可以成为gvt持有者。

- 2. **阈值$\tau _{l}、\tau _{m}$**
D2FQ通过在虚拟时间内节流快速流(即具有低权值的流)来调节公平性。两个阈值(τm和τl)是何时抑制这种快速流动的标准。
![在这里插入图片描述](https://img-blog.csdnimg.cn/69a17f3a9d374bc8a34b177ed8f0c80f.png)
这些流平均共享I/O带宽的时间随着τl增大而增大，且可以看出最终共享带宽相同。延迟时间与阈值并没有呈线性关系。
![在这里插入图片描述](https://img-blog.csdnimg.cn/15584117b7384dcca7c099540c05a84d.png)
当阈值为1 MB时，相比其他两个阈值，此时三个流显示出3.7倍的长尾延迟。
对于工作负载的特性，特别是I/O大小和流的权重，会影响阈值的选择，因为I/O大小和权重决定了虚拟时间增加的幅度。
D2FQ使用了一个经经验发现适合所测试工作负载的适当τl。将阈值的微调留给用户或系统管理员。此外，根据经验，取τm = τl/2为宜。

- 3. **H/L ratio 与中等队列权重**
H/L比值为高队列权重与低队列权重之比，它是满足公平性的最重要因素。中等队列权重是H/L比值的平方根与低队列权重的乘积，这样可保证高队列和中队列的速率比等于中队列和低队列的速率比。
D2FQ收集所有流的虚拟时间信息及其I/O使用统计信息，例如队列和I/O大小。它利用收集到的信息周期性地寻找合适的H/L比。
1. 当不满足公平性时，H/L比率需要增加，缩小他们vt之间的差距。τw是发现不公平现象并触发寻找合适H/L比值的阈值。
D2FQ会跟踪最大虚拟时间流和最小虚拟时间流，通过计算上一周期的虚拟时间的增量来确定新的H/L。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4304f1ba92874f49892ced362e6a53c6.png)
2. 但是H/L比值太大会增加需要节流的流的尾延迟。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4952dd8b3e294dbaab86ed7f9dd3ad52.png)
高H/L比会使fb使用high queues，出现与fa争抢，并且它的带宽会降低，导致延迟增加。
因此，有时候也需要降低H/L。但H/L比减小的条件是最大虚拟时间差小于τw。
D2FQ使用上一个统计收集周期的I/O统计信息计算每个流的虚拟减速，虚拟减速是该流的I/O请求在不使用高队列时的慢度估值。
![在这里插入图片描述](https://img-blog.csdnimg.cn/e2961bce08a442b996646be584504ea2.png)
$\sum l_{f},x$为上一周期流f提交给队列类x的I/O总量，$\frac{p_{x}}{p_{y}}$为当前两个队列类x和y的权重比。
然后，D2FQ在系统所有活动流中选择最大的虚拟减速，并设置下一个H/L比为选出的最大值。

### 4. 实验
![在这里插入图片描述](https://img-blog.csdnimg.cn/9060992b9fc644c39087eee1194e3429.png)
> 四种调度器：
-None，在块层不进行I/O调度
-D2FQ ，论文的方案，设备端调度
-MQFQ，最先进的公平排队I/O调度器，在块层中进行调度
-BFQ，Linux中基于时间片的比例共享I/O调度器

1. **公平性的对比**
![在这里插入图片描述](https://img-blog.csdnimg.cn/94519e2169a9416a821e5bfecf6ae7d1.png)

2. **动态调整H/L**
构建一个包含三个具有不同生命周期的流组的合成工作负载：（1）包含一个weight-1流并从头到尾运行的基础，（2）包含三个weight-3流并从10秒时间点运行到结束的事件1，以及（3）包含四个weight-2流并从20秒时间点到结束运行的事件2。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f11b5efb087a4623a9d369b5e455c9fc.png)

3. **现实工作负载**
FIO更需要带宽；YCSB，它每I/O消耗的CPU周期比FIO多。
![在这里插入图片描述](https://img-blog.csdnimg.cn/bb3b67da7e7a4936a1502aeaf2e9ac14.png)

从图18可以看出D2FQ和MQFQ都具有公平性，由于FIO与YCSB占用的带宽不同，那么YCSB（更消耗CPU）占比大的那么CPU利用率会更大。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f782b6cd868e4087990cf330071f9fe8.png)
D2FQ和MQFQ的FIO带宽和YCSB带宽均小于None。因此，两者都显示FIO的I/O延迟更长，YCSB的I/O延迟更短。由于BFQ在I/O调度方面效率低下，它显示出很长的I/O延迟。

问题：
1. 为什么WRR中没有考虑ASQ?
2. 提交即分配，但是D2FQ里也需要计算虚拟时间并与阈值比较，这不相当于仲裁吗？
