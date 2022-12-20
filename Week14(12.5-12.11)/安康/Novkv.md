NovKV: Efficient Garbage Collection forKey-Value Separated LSM-Stores
Chen Shen, Youyou Lu∗, Fei Li, Weidong Liu∗, Jiwu ShuTsinghua University
MSST'20（B刊）
Conclusion:
键值分离方法必须通过查询LSM树来检查键值项的有效性，并通过在垃圾收集期间将其插入回LSM树中来更新value handles，本文通过消除LSM树的查询和插入来减少垃圾收集的开销。该方法包括三个关键技术：collaborative compaction, efficient garbagecollection,  and  selective  handle  updating. LevelDB上实现了这种方法，并将其命名为NovKV。

