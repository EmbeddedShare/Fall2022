### REMIX：Efficient Range Query for LSM-trees

Fast21 Wenshao Zhong 

![image-20221018182142425](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210181821465.png)

Conclusion：

本文借助 sorted view 为 LSM-tree 查询构造持久化的索引（未采用最小堆策略），segment内部使用二分查找，但缺点是REMIX 重建占用了较多读 IO

**Introduction：**

LSM缺点：

1）冗余存储，对于某个key，实际上除了最新的那条记录外，其他的记录都是冗余无用的，但是仍然占用了存储空间。因此需要进行Compact操作(合并多个SSTable)来清除冗余的记录。

2）读取时需要从最新的倒着查询，直到找到某个key的记录。最坏情况需要查询完所有的SSTable，这里可以通过前面提到的索引/布隆过滤器来优化查找速度。

LSM树的Compaction策略：

1) size-tiered 策略：保证每层SSTable的大小相近，同时限制每一层SSTable的数量，当每层SSTable达到N后，则触发Compact操作合并这些SSTable，并将合并后的结果写入到下一层成为一个更大的sstable。

缺点：当层数达到一定数量时，最底层的单个SSTable的大小会变得非常大。并且size-tiered策略会导致空间放大比较严重。即使对于同一层的SSTable，每个key的记录是可能存在多份的，只有当该层的SSTable执行compact操作才会消除这些key的冗余记录。

2)leveled策略也是采用分层的思想，每一层限制总文件的大小。

但是跟size-tiered策略不同的是，leveled会将每一层切分成多个大小相近的SSTable。这些SSTable是这一层是全局有序的，意味着一个key在每一层至多只有1条记录，不存在冗余记录。读操作高效，写效率变低。

范围内的键可能位于不同的表中，这可能会因为高计算和I/O成本而降低查询速度。

**Background&Motivation：**

minor compaction：当MemTable填满时，缓冲键将被排序，并作为排序运行，由一个称为轻微压缩的进程刷新到持久存储。因为更新是按顺序分批写入的，而不与存储中的现有数据合并

minor compaction：Minor Compaction 是一个时效性要求非常高的过程，要求其在尽可能短的时间内完成，否则就会堵塞正常的写入操作，因此 Minor Compaction 的优先级高于 Major Compaction。当进行 Minor Compaction 的时候有 Major Compaction正在进行，则会首先暂停 Major Compaction。

Seek查找--大于等于的最小值，每一级level都要查找

**稀疏索引**：fast09索引中仅针对表中部分数据建立索引项，具体说，就是为表中每个数据块（block）建立一个索引项。 例如：表中索引列值为1,2…10的十条数据，而索引中只为索引列值为1,5,9的数据创建相应的三个索引项，其中，索引项1指向 1,2,3,4这四条数据所在的表中数据块。

**Cursor offsets:**

REMIX 将 sorted view 涉及到的 key range 按照顺序划分为多个 segment，每个 segment 内保存一个 anchor key，记录该 segment 内最小的 key。

此外，每个 segment 内还维护了 cursor offsets 和 run selectors 两个信息。Cursor offsets 记录了每个 run 中首个大于等于 anchor key 的 key offset，比如 11 这个 segment，run 1,2,3 首个大于等于该值的 key 是 11, 17, 31（offset 1, 2, 1）。Run selectors 顺序记录了 segment 内每个 key 所在的 run 序号，以 71 这个 segment 为例，71, 73, 79 分别在 run 0, 1, 0 中。

每个cursor offset记录了每个run中等于或者大于anchor key的最小key在run中的offset，

查找键值：

![image-20221018182225982](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210181822023.png)

假设点查 17，基于 REMIX 可以进行如下的查询过程：

通过 anchor key 二分查找，定位到 17 落在第二个 segment 的 key range 内。

由于 cursor offsets 代表着各个 run 中首个大于等于 anchor 的 key，17 > 11，所以直接将它作为各个 run 的初始 cursor，即 (1, 2, 1)。

将当前 pointer 设置为 run selectors 中的第一个，即 run 0，相当于原先的最小堆 top。

从 pointer 指向的 run 开始比较，11 (run 0) < 17，所以推进 run 0 的 cursor，即 offsets 变为 (2, 2, 1)。同时，run selectors 也向前推进，由 run0 切换至 run1，表示下一个需要访问的是 run1 在 cursor offsets 中对应的 key，即 run1 中下标为 2 的 key（17）。

查询结束。

范围查询过程：不断地根据 run selectors 调整访问的 run 及 offset，并结合与 end key 的比较结果，即可将点查过程扩展为范围查询。

 

**Efficient Search in a Segment**

 

![img](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210181818077.jpg) 

本文借助了 run selectors 来进行二分查询，但由于 selector 仅记录了该 key 归属的 run，并不确定是该 run 中的哪一个 key，所以此处还计算了一份 occurrences 来辅助定位。该值表示的是当前 selector 所在的 run 在 segment 内是第一次出现，通过该 run 的 cursor offset 加上 selector 对应的 occurrence 值，即可得到它在该 run 中对应 key 的下标。

以上图 16 个 selector、查询 41 为例（假设 cursor offset 都是 0）：

访问第 8 个 selector，通过 selector 和 occurrence 确认该 key 为 run3 的第 4 个 key（43），>41，所以向左查询。

访问第 4 个 selector，run2 第一个 key（17），<41，向右查询。

不断二分，最终查询到 41 为 run3 的第 3 个 key。

显然，不管是基于 selectors 的二分还是基于 occurrences 快速跳过 run 内元素，都能有效地减小 key 之间的比较次数，降低计算的开销。

![img](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202210181818692.jpg) 

作者基于 REMIX 实现的一个 KV 存储，由于 workload 往往具有很强的空间分布特性，RemixDB 使用了 partitioned store layout 以降低 compaction 的开销。

Partitioned store layout 和分布式存储中基于 partition 划分数据的思想类似，将不同 key range 的数据分布到不同的 partition，从而每个 partition 可以独立地进行 compaction，相互之间不产生影响。RemixDB 中的每个 partition 都可以视为传统 tiered compaction LSM-tree 里的一个 level，即其中包含多个 run，此外为该 partition 内的所有 run 维护了一个 REMIX 索引。