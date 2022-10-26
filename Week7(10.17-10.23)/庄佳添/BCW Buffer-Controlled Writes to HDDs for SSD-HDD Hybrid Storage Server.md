## BCW: Buffer-Controlled Writes to HDDs for SSD-HDD Hybrid Storage Server

这是阿里在fast20上的一篇文章，第一作者为Shucheng Wang。作者主要针对当前混合存储系统下对SSD和HDD利用不均衡的问题，提出一种缓冲区控制的写方法BCW，利用HDD写入延迟存在fast、mid、slow的周期性，尽可能的在fast、mid阶段以us级的延迟将用户数据写入HDD，在slow阶段填充非用户数据。并基于BCW，设计了一个混合IO调度器MIOS，根据写模式、运行时队列长度和磁盘状态自适应地引导传入数据到SSD和HDD。

### Background

SSD和HDD混合存储中，SSD通常用作数据持久化的缓存，之后再将数据刷入HDD中，因此SSD吸收了上层应用的大部分写入流量。阿里云的盘古存储系统采用混合存储方式，如Table 1所示，作者对盘古生产环境的使用情况观察后发现系统对HDD和SSD的利用上存在明显的不均衡现象，SSD被过度使用而HDD的利用率较低。作者分析盘古存储节点上的四种典型负载：（A）计算型任务；（B）存储型任务；（C、D）结构化存储，得到如下特点：1. 大多是请求位写请求；2. 如Figure 2所示SSD经历突发密集型写入 ；3. SSD和HDD写入量差异巨大 ；4. 如Figure 2a所示存在长尾延迟(第99个百分位延迟位10ms)，原因为如Figure 2b所示SSD队列阻塞；5. 小体积IO占大多数。

![image-20221017201556947](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210172016243.png)

通过对HDD的实验作者发现，HDD的写入延迟存在周期性，有fast、mid、slow三个阶段。由Figure 1可以看出fast和mid阶段HDD写入延迟再us级，接近SSD的水平，这是因为HDD会在其内置的DRAM中给写请求分配一块buffer，当将请求数据写入buffer后HDD会向上层返回成功。而当buffer中的数据狼达到一定阈值时，HDD需要将buffer中的数据刷入硬盘中，这个过程阻塞后续的写入，导致延迟的增大。也可以显示的通过sync()来将buffer数据刷入磁盘。

### Motivation

在写负载持续增高的情况下，以 SSD 为主的混合存储系统会面临如下问题：

1. SSD 寿命：持续的高负载会缩短 SSD 寿命，从数据上来看，SSD 的每日写入情况经常触及所设上限，即 DWPD (Drive Writes Per Day)。
2. SSD 性能：在写入密集的情况下，大量的写请求可能会超过 SSD 的处理能力，导致请求在 SSD queue 中排队，引起长尾延时；除此之外，请求的增加也会更频繁地触发 SSD 的 GC，导致性能下降。

解决该问题最简单的方法是引入更多的SSD，但这无疑会增加额外的硬件成本。现有的将对SSD的写请求重定位到HDD的方式极大增加系统的响应延迟。因此主要的挑战是如何将HDD的延迟降到us级别。通过Background部分作者对HDD写入周期性的发现作者得到一个解决思路：**预测HDD下一次的写入状态(fast、mid或slow状态)，在HDD能够提供us级延迟时，将写入请求交给HDD处理。**



### An HDD Buffered-Write Model

为了更好地研究HDD写入延迟周期性这一特性，作者对此进行了建模，如Figure 4所示，模型参数如Table 2所示。F、M、S分别对应fast、mid、slow阶段。一次序列写入持续一个fast阶段，接着slow和mid阶段接交替。其中$W_{f}=T_{f} * s_{i} / L_{f}$计算1个fast阶段可以写入的数据量大小，其他阶段同理。																																																			

![image-20221019100643880](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191006259.png)![image-20221019100926696](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191009718.png)



### Design

作者设计**Write-state Predictor**来预测下一次写时HDD所处的阶段(fast mid slow)。Write-state Predictor根据**当前buffer剩余空间和当前写状态**来预测。buffer状态分为Available和Unavailable，分别对应累计数据写入量ADW小于和大于$W_{f}$和$W_{m}$的情况。状态装换图如Figure 5所示。伪代码如Algorithm 1所示。



![image-20221019165509679](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191655728.png)

![image-20221019170004255](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191700320.png)

Buffer-Controller writes(BCW)是一种确保用户写在F或者M阶段写入磁盘的HDD写入方式。其伪代码如Algorithm 2所示。激活 BCW 后，将调用 sync() 操作以强制同步主动清空缓冲区。如果预测为 F 或 M 状态，BCW 将顺序用户写入调度到 HDD，否则填充到 HDD直到S状态结束。如果队列中有用户请求，BCW 将它们串行写入。写入完成后，BCW 将其写入大小添加到 ADW。 BCW在写请求稀疏时也使用填充，两种类型的填充数据，PF和PS。PF数据较小，用于在S和M阶段使用，减少用户请求的等待时间，PS为大块数据帮组更快结束S阶段。

- PS padding：由于预测算法会在 F/M 状态下的 ADW 接近 Wf 或 Wm 时返回 S 状态，BCW 根据此可以得知，buffer 即将被填满，所以它通过主动地构造 PS padding 数据（较大，64KB）来触发 slow 写入，直到某次写入的时延对应的状态不再为 S，BCW 即认为当前 HDD buffer 以恢复到能够以微秒时延写入数据的状态，它会重置 ADW。
- PF padding：考虑到低负载的情况下，HDD 可能不会收到任何写入请求（可能 SSD 足够处理），为了保证算法的稳定性，BCW 会在非 S 状态时不断写入 PF padding（较小，4KB）。算法中仅在预测状态为 M 的情况下进行此操作，这是因为当 sync 或者 HDD 内部隐式 flush 被执行后，buffer 会进入到稳定的 F 状态，此时无需做任何的 padding。

![image-20221019171050615](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191710655.png)

Mixed IO scheduler(MIOS)的结构如Figure 6所示，MIOS为一个调度器，根据HDD/SSD的状态进行综合调度，决定每一个用户写入请求由SSD还是HHD处理。调度算法如Algorithm3所示，其主要逻辑如下：

- 当 flag 被设置时，HDD 一定处理 S 状态，此时请求只能由 SSD 处理。
- 如果 SSD 并不忙（队列长度 l(t) 并非超过设置的阈值 L），当 HDD 处于 F/M 时，交由 SSD 处理对性能最好。
- 当SSD忙且HDD不处于S状态可由HDD处理。

在基本逻辑之上，调度算法还被细化为 `MIOS_E` 和 `MIOS_D`，两者的区别在于当 SSD 不忙且 HDD 处于 F 状态时，前者会将请求转发给 HDD 以进一步地降低 SSD 的负载。

需要注意的是，MIOS 算法需要拥有对 HDD 的完全控制，所以当读请求到来时，BCW 算法会被挂起来处理此请求，此时不能再向该 HDD 写入数据。这也比较容易理解，当读请求到达时，HDD 的磁头可能就跑到了另外的地方，无法再保证连续写的要求。因此，对于 read-dominated workload，MIOS 并不适用。

![image-20221019184307077](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191843124.png)

![image-20221019184825445](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210191848492.png)

### Evaluation

##### Production Workloads

论文使用了前述的 4 种 workload 对 MIOS 算法进行了详尽的实验，结果如下。

![image-20221026211225928](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262112978.png)

时延对比：无论是平均时延还是长尾时延，MIOS 都拥有更好的效果。

![image-20221026211433575](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262114613.png)

SSD 队列长度分布也体现了长尾延时的降低。

![image-20221026211246838](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262112876.png)

不同请求大小下的平均时延：对于大请求，MIOS 的效果比 baseline 更差，一方面是在写入大请求时，SSD 本身比 HDD 拥有更佳的性能（内部并行机制），另一方面则是大请求相对较少，被 SSD queue length 或 GC block 的概率也较低。

##### MIOS_E vs MIOS_D

![image-20221026211309328](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262113366.png)

![image-20221026211319260](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262113301.png)

因为 MIOS_E 允许在 SSD 不忙的情况下将请求转发给 HDD，所以相比 MIOS_D，它会转发更多的请求，但也会导致时延上升。这个现象对于 workload A 特别明显，从表 3 可知，相比其他三个 workload 而言，它对 SSD 的 workload 很低，这也使得在 MIOS_D 下，大部分请求仍旧由 SSD 进行处理，能够获得更好的性能，但在 MIOS_E 下，请求被转发给 HDD，导致了性能下降。

但这并不意味着 MIOS_E 毫无用武之地，当 SSD 的写入性能本身就一般的情况下，即使它的 queue length 并未表现出忙的特征，但实际写入的延时可能依旧较高，此时转发给 HDD 反而能获取更好的性能。作者尝试将 SSD 替换为 660p（原先为 960EVO，性能更佳）后，MIOS_E 表现非常好。

![image-20221026211341529](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262113568.png)

除了性能以外，因为 MIOS_E 会收到更多的 HDD 请求，从而算法中的 padding 数据也会增多，所以它相比 MIOS_D 会产生更多的空间浪费。另外，MIOS 算法将部分 SSD 负载搬迁到 HDD 上执行，会有效提高 HDD 的利用率，但仍需要确认：**HDD 仍有足够能力来承担数据搬迁（SSD->HDD）任务。**实验对此进行了验证，有兴趣的同学可以参考原文，此处不再赘述。

##### Write Intensity

![image-20221026211407608](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210262114641.png)

由于 MIOS 利用了 HDD 连续写的特性，所以它非常适合 write-intensive workload，作者对此进行了补充测试（X 轴代表的是发送间隔，越小数据量越大）。

可以看到，当写压力很大的情况下（20-60us），SSD 的性能会受到排队和 GC 的影响，平均时延和长尾时延都要高于 MIOS。而当压力降低到可承受范围后，SSD 将保证稳定的写入性能，此时，MIOS_D 退化为纯 SSD 写入（因为 SSD 无忙特征），但 MIOS_E 依旧会转发部分请求至 HDD，所以相对之下会有更高的平均和长尾时延。



### Conclusion

在写为主的工作负载下，SSD和HDD混合的存储结构导致SSD过度使用而HDD没有得到充分利用。HDD在fast和mid周期能够实现μs级别的写IO延迟的特性没有被利用起来。因此作者提出了一种缓冲控制的写方法BCW，将用户请求在fast和mid阶段下发到HDD，而在slow阶段填充非用户数据，主动控制缓冲写，通过请求重定将写请求从过度使用的SSD上卸载写请求到HDD。并基于BCW，设计了一个混合IO调度器MIOS，根据写模式、运行时队列长度和磁盘状态自适应地引导传入数据到SSD和HDD。

