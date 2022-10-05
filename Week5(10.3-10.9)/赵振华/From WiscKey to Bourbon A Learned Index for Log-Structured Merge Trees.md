## From WiscKey to Bourbon: A Learned Index for Log-Structured Merge Trees

本文由威斯康辛麦迪逊大学 Remzi指导并发表于OSDI 2020。以往的大量研究表明可学习索引能够提升基于树（如B+tree）的数据访问性能。本文分析了LSM-tree的操作特点，通过分析不同负载下LSM-tree各层/SStable的状态变化总结在其内部使用可学习索引的5个指导思想。作者选用贪婪分段线性回归（Greedy-PLR）在WiscKey基础上构建了Bourbon并在File和Level粒度上对比了使用可学习索引的优缺点。在人造数据集以及真实数据集上的实验均表明Bourbon可以有效提升LSM-tree的查询性能。

### Background

在数据库系统中引入可学习索引的思想早已出现。为了查找指定key，系统可以使用已经学习好的函数来预测key（以及value）的位置，如果预测成功将大大提升系统的查询性能。以往工作尚未聚焦LSM-tree，本文试图将可学习索引的思想引入LSM-tree。可学习索引通常是为只读场景定制的，当数据发生写操作时，在其上已经学习到的模型必须做出调整以适应这种变化。虽然LSM-tree具有写优化的特点，但是写入操作只会改变LSM-tree的一部分，所以树的大部分数据处于相对稳定的状态。鉴于此，LSM-tree仍然可以利用可学习索引来提升其读性能，但需要处理可变长的key/value以及平凡训练模型带来的资源浪费等挑战。

#### LSM、LevelDB and WiscKey

LSM-tree采用顺序写实现高写吞吐。key-value对首先被写入内存中的memtable，当memtable写满将转变为immutable table，随后被flush到磁盘组织成SStable（每个对应一个文件），磁盘中的SStable大小超过限制后会激发Compaction操作，当前level中的多个SStable将和下一层的SStable合并后会写入下一层。下图展示了对于关键字 $k$ 的查询操作流程，作者将所有的七个步骤分为两类：**Indexing step**包括①（查找候选SStable）③（查询Index block）④（查询Bloom filter）⑥（Binary Search data block）。**data-access step**包括②（加载索引块和布隆过滤器块）⑤（加载数据块）⑦（读取数据块中的key-value）。本文旨在减少**Index step**的时间。

在LevelDB中，因为key和value被一起排序和重写，compaction操作会导致较大的写放大，因此LevelDB存在较大的合并开销，这会影响前端工作负载。为了减小这一情况，WiscKey将value单独存放，SStable只保存key以及指向value的指针，即合并操作只面向key，value不发生变化，从而减少了I/O放大和性能开销。WiscKey的读写路径和LevelDB相似，只是value被写入value log。WiscKey的LSM-tree比LevelDB小的多，可以完全放入内存中，因此一次查询涉及多次内存搜索。

#### Optimizing Lookups in LSMs

利用可学习索引优化LSM-tree的核心思想是在输入数据的基础上训练模型来预测有序数据集中记录的位置。若查询过程中预测正确，则返回需要的记录，如果预测失败，将在误差范围内进行局部搜索。例如预测的位置为$pos$,最小最大误差边界分别为$\delta_{min}$、$\delta_{max}$，则如果预测错误，局部搜索的范围为：$pos-\delta_{min}$到$pos+\delta_{max}$之间。在此基础上有两个问题需要回答：（1）可学习索引可以加速LSMs的查询操作吗？（2）如何在维持LSMs写优化的性能下发挥可学习索引的性能优势？

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004091400618.png" alt="image-20221004091400618" style="zoom: 50%;" />

#### Learned Indexes

优化索引例如使用可学习索引可以减少查询过程中的indexing时间，但是不能降低data-access时间，所以如果查询过程中数据索引过程占比较大，可学习索引降低查询延迟的潜力还是巨大的。尤其是当数据集或其中的一部分被缓存在内存中时，data-access时间是低的，此时indexing开销占比较大。下图为WiscKey中查找延迟的分解，可以看到InMemory的情况下可学习索引具有优化潜力，随着使用的存储设备越来越快，Indexing占比越来越大，可学习索引潜在的优化能力也逐渐增大。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004091619204.png" alt="image-20221004091619204" style="zoom: 50%;" />

可学习索引在面对write-heavy负载时学习的模型必须时常更新，这将增加系统负载。对于LSMs来说更新操作下大部分数据没有发生改变，新更新的项被缓存在内存中或保存在较高的level中，低level中的数据保持稳定。由于大部分数据都较稳定的保存在低level中，所以涉及的重复学习也很少，相反高level中的数据变化速度较快，使用可学习索引时的高训练率将拉低收益。作者也提到了SStable的不变性使其可以成为训练索引模型的良好单位，此外level也可以被整体学习。

### Exploratory Experiment

为了更好的在LSMs中使用可学习索引，作者进行了下列探索性实验：

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004091717976.png" alt="image-20221004091717976" style="zoom: 80%;" />

首先是分析各层SStable的生存周期以反应模型的有效期限。作者记录了所有SStable的创建和删除时间，从上图a中可以看到（1）低层SStable的生存周期要比底层高的多（2）在低写负载下，高层SStable的生存周期也很长（3）虽然SStable的生存周期随着写操作比率的增加而降低，但是在低层SStable的生存周期仍然很长。上图b、c展示了L1层和L2层SStable生存周期的累积分布函数。在L1层，大概50%的SStable生存周期只有大约2.5sec，当越过这个阈值，file将存活更长的时间（大约5min）。在L4层虽然大部分SStable文件的生存周期很长，也有大约2%的文件生存周期不超过1sec，这和合并操作的逐层下推有关。通过以上观察，作者总结前两个用于可学习索引的guidelines：

- 底层SStable生存周期更长，更适合使用可学习索引；
- 即使在底层，SStable生存周期也可能很短，因此当文件被创建后最好等待一段时间再开始学习；

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004091827860.png" alt="image-20221004091827860" style="zoom: 67%;" />

为了衡量一个索引模型被使用的次数，作者衡量了每个SStable文件服务的查询数量。作者统计了每一层中SStable文件服务的查询数量并计算平均值，数据集是以正太随机顺序加载的（uniform random order）。上图a展示了统计结果。虽然大部分数据都驻留在LSMs底层，高层的内部查询却更高。可以看到高层LSMs中失败的内部查询更多，而底层LSMs中成功的内部查询更多。这表明高层LSMs中即使Bloom filter已经使得查询更快，但是仍然需要搜索Index Block。上图b展示了当数据集被顺序加载时（例如key按升序插入）的实验结果，和随机加载数据不同此时没有失败的内部查询因为各层级之间的SStable不会重叠，底层SStable服务了更多的查询因此可以从可学习索引中获益更多。由此作者总结了两个guidelines：

- 不要忽视上层SStable，虽然低层SStable生命周期更长，高层SStable任然涉及许多内部查询；
- 经量做到对工作负载以及数据具有感知性，虽然大部分数据驻留在低层，但是如果查询不涉及低层，那么可学习索引也不会带来性能提升，因此学习必须具有工作负载可感知性。此外数据加载的顺序会影响那一层SStable接收更多的查询，因此系统必须具有数据感知性；依据内部查询的数量，系统需要动态决策是否对一个文件进行学习。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004091912093.png" alt="image-20221004091912093" style="zoom: 50%;" />

作者接着统计了各level的生存周期如上图所示。L0层数据具有无序性，不适合进行学习。当有新的SStable被创建或者删除，Level将发生改变，因此模型需要重新学习。以level为粒度有好有坏，坏处是学习的频率较高，好处是查询直接返回候选SStable以及内部记录的偏移量。上图a展示了在5%写入负载下每层file变化次数与该层总文件数量之比随时间的变化。只要变化为0，该层学习的model便可被利用。我们除了可以观察到低层文件变化较少，还可以看到变化是突变的，这是由于合并操作导致一层的许多SStable被重写，并且这一突变在各层之间具有一致性。由此作者总结了第五个guideline：

- 在高写入负载下，不要以level为粒度使用可学习索引；

### Solution and Implementation Detail

对于索引的学习作者使用了PLG（分段线性回归），其具有较低的时间成本与空间成本。PLR背后的思想是用一系列的线段表示有序数据集，为了训练PLG，BourBon使用了贪婪算法，每次只处理一个数据点，当处理的数据点在满足误差边界的前提下无法加入当前线段，则创建一条新线段。一旦模型被创建，可使用二分查找找到需要的线段，然后用key乘以斜率再加上截断得到预测的位置。查找的复杂度为$O(logs)$，其中$s$为线段数量。BourBon要求key是固定大小的（若不同可使用padding），value大小可以变化（借鉴WiscKey的思想）。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092043290.png" alt="image-20221004092043290" style="zoom: 50%;" />

对于可学习索引的学习粒度，作者做了上图实验比较了不同负载下的性能特点，可以看到在read-only负载下使用level粒度的可学习索引是高效的。考虑到BourBon是在提供写操作的情况下使用可学习索引因此level粒度的模型训练不是好的选择，所以BourBon默认采取file粒度的可学习索引。

#### Cost vs. Benefit Analyzer

为了确保花费在索引学习上的时间是值得的，BourBon采用在线代价收益分析机制。 为了避免学习生存周期较短的file，BourBon在学习一个文件之前会等待一个时间阈值$T_{wait}$，该参数反应了代价和性能之间的权衡，作者将这一参数设置为学习一个file需要的时间，事先通过测算，具体设置为50ms。考虑到有些生存周期较长的文件涉及的查询很少，而高层文件虽然生存期短，但是在某些情形下涉及较多的失败内部查询。因此BourBon必须权衡学习一个模型的性能收益。当模型收益$B_{model}$超过训练代价$C_{model}$则这个模型是正收益的。

- **估计$C_{model}$：**如果后台有足够的空闲cpu，则可忽略不计，保守起见，设置为$T_{build}$，即为一个文件训练PLR的时间，该时间与文件中数据的数量成正比。$T_{build}$等于文件中数据量乘以训练一个数据点花费的时间（离线测量）。

- **估计$B_{model}$：**作者将内部查询分为成功和失败两种情况，规定$B_{model}=(T_{n.b}-T_{n.m})*N_{n}+(T_{p.b}-T_{p.m})*N_{p}$，其中$N_n$、$N_m$分别为一个文件涉及的失败内部查询和成功内部查询的数量，$T_{n.b}$、$T_{p.b}$分别为baseline下失败查询以及成功查询的时间。在不知道上述查询数量和所需时间的条件下我们无法计算$B_{model}$，因此分析器记录了文件在生命周期内的统计数据，例如创建/更新时间、涉及多少查询等等。为了估计文件F的这些信息，分析器使用了同一层其他文件的统计信息（跨层之间这些信息差距很大）。

  - BourBon在对文件进行学习前需要等待一段时间，这段时间查询通过原有路径处理，BourBon利用这段时间估计$T_{n.b}$以及$T_{p.b}$；
  - $T_{n.m}$以及$T_{p.m}$的估计通过计算同一层其他文件的平均值估计；
  - 对于$N_n$以及$N_p$，首先计算同一层其他文件成功/失败查询平均数，然后乘以比例尺$f=\frac{s}{\overline{s_l}}$,其中$\overline{s_l}$为同一层文件大小的平均值；

  当刚启动时，系统没有足够的统计数据，此时BourBon运行在学习状态，当积累足够多的统计数据后，BourBon将运行分析器比较学习一个文件的$C_{model}$以及$B_{model}$，如果收益成正比，则进行模型训练。

引入可学习索引的BourBon查询流程如下所示，对于系统已学习的SStable涉及的查询可以通过以下流程处理：

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092147484.png" alt="image-20221004092147484" style="zoom: 50%;" />

### Evaluation

实验环境：Intel Xeon CPU E5-2660、160-GB memory 、480-GB SATA SSD、16B integer keys and 64B values、PLR error bound as 8。数据集分为线性数据集、1%分段、10%分段和正态数据集，每个数据集都有64M的键值对，详细信息参考原论文。大部分分析情况数据都放在内存中。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092228984.png" alt="image-20221004092228984" style="zoom: 50%;" />

- **Which Portions does BOURBON Optimize?**

  我们看到搜索时间以及数据加载时间都降低了，数据加载时间降低是因为BourBon减少了需要加载的数据量，在baseline中需要加载整个Block。

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092246525.png" alt="image-20221004092246525" style="zoom:50%;" />

-  **Performance under No Writes**

  分析当模型已经建立并且数据库没有更新操作的情况

  - **Datasets**

    可以看到在线性数据集下BourBon提供最好的性能提升，因为线性数据集下训练获得的模型线段最少，搜索目标模型线段的时间最少。在只读负载下BourBon-level对baseline的性能提升也很可观，其余实验聚焦file粒度的可学习索引。

    <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092259627.png" alt="image-20221004092259627" style="zoom:50%;" />

  - **Load Orders**

    顺序加载的数据集中加速性能较好，应为在顺序加载下没有失败的内部查询，而在随机加载的数据集中存在失败的内部查询。虽然BourBon同时加速了positive查询和negtive查询（有没有最终查到），但是对失败的内部查询加速有限，因为大部分失败的内部查询在查询filter之后就结束了，后续不会加载data block并进行搜索，考虑到失败的内部查询数量更多，所以BourBon加速有限。

    <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092442150.png" alt="image-20221004092442150" style="zoom:50%;" />

  -  **Request Distributions**

    为了分析不同请求负载下的性能，首先随机加载数据集然后运行负载，可见在六种负载下BourBon均可加速查询。

    <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092453449.png" alt="image-20221004092453449" style="zoom:50%;" />

- **Range Queries**

  当查询范围较短，索引过程占据重要部分，BourBon可以很好的加速，但是随着查询范围的变大，索引开销（定位第一个key的位置）占比越来越小，此时模型加速的效果不是很明显。

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092507683.png" alt="image-20221004092507683" style="zoom:50%;" />

-  **Effificacy of Cost-benefifit Analyzer with Writes**

   测量BourBon在有写入操作负载下的性能表现，此时对cost-benefit analyzer（cba）效用要求较高。

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092534104.png" alt="image-20221004092534104" style="zoom:80%;" />

  上图展示了具有不同写操作比例的负载下系统的性能表现，一共有三种策略cbd、offline以及always。BourBon-offline不进行学习，模型只存在与最初加载的数据，BourBon-always每次数据写入时都对进行重新学习，BourBon-cba使用cost-benefit analyzer来判断是否需要对文件进行学习。实验结果表明，所有的BourBon策略都降低了WiscKey前端查询以及写入时间总和，随着写操作比例的增加性能收益降低，因为系统没有足够多的查询可以优化。BourBon-always策略比较激进，随着写操作的比例逐渐增大用于学习的时间也逐渐增大，总的时间开销最终超过了baseline，因此激进的学习策略是不可取的。对于具有较低比例写操作的工作负载，BourBon-cba决定对大部分文件进行学习，但是随着写入增加其意识到收益降低，减少了学习时间。从图d中也可以看到BourBon-cba在模型训练成本和收益之间进行了trade-off，当写入负载增多时越来越多的查询通过原有路径完成。

- **Real Macrobenchmarks**

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092554976.png" alt="image-20221004092554976" style="zoom: 80%;" />

  <img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092617061.png" alt="image-20221004092617061" style="zoom:50%;" />

- **Performance on Fast Storage**

   在Intel Optane SSD上的实验结果

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092707552.png" alt="image-20221004092707552" style="zoom:50%;" />

- **Performance with Limited Memory**

  在拥有SATA SSD并且内存仅可容纳25%数据集的情况下的实验结果。在正态分布工作负载下，Bourbon的速度只快了1.04×，因为大部分时间都花在了将数据加载到内存中上。相比之下，在zipfian工作负载中，因为大量请求访问的是已缓存在内存中的小部分数据集，索引时间而不是数据访问时间占据了主导地位。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092723201.png" alt="image-20221004092723201" style="zoom:50%;" />

- **Error Bound and Space Overheads**

  error bound参数设置为8是通过实验获得的。

<img src="https://raw.githubusercontent.com/Smartog/picturebed/master/Bourbon/image-20221004092747488.png" alt="image-20221004092747488" style="zoom:50%;" />

### Conclusion

当模型已经建立并且没有写入时，BourBon为各种数据集、加载顺序和请求分布提供了比基线显著的速度。Bournon-always激进的学习策略可以提供快速查找，但成本很高。Bournon-offline加速效果不好，两者都不是理想的，相比之下，Bourbon-cba在两者之间进行了权衡，具有数据感知性。这篇论文引申出的未来工作包括：使用其他不同的学习模型和算法、在计算$B_{model}$和$C_{model}$时可以统计更加复杂的数据例如根据key的语义、在level粒度以及file粒度之间进行模型学习的动态转换。



