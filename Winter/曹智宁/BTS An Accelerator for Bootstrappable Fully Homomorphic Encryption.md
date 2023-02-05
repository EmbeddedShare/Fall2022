## BTS: An Accelerator for Bootstrappable Fully Homomorphic Encryption

### Abstract

背景：同态加密使得云计算数据安全。同台加密基于噪声加密方案，但是随着计算噪声会逐步增大，这使得计算不能一直进行，所以需要bootstrapping操作通过刷新密文来解除限制。但是bootstrapping操作需要大量的附加计算和内存带宽。

贡献：前期工作提出了硬件加速全同态加密计算，但是我们的工作做出了第一个全同态加密的硬件。我们提出了用于全同态加密的BTS（ Bootstrappable, Technology-driven,Secure)加速器架构。

Challenge：在加速器上支持bootstrapping，分析片外内存带宽和计算需要。考虑到现在内存技术的限制，确定了有效加速的同态加密参数

总结：我们实现了并行，给出了设计和微结构。速度比CPU提升$5556$倍和$1306$倍，板片区域大小为$373.6mm^2$功率最高为$163.2W$

### 1 Introduction

同态加密在机器学习云计算加密方面有很大贡献，但是有噪声随计算增大的问题。

同态加密主要的问题在于高计算量和高内存带宽需要。近期有新的计算方案和新的硬件加速方案。但是最近的加速器方案只关注N很小的情况，且缺少对bootstrapping的支持。table 1为近期加速器展示。

<img src="C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Tabel1.png" alt="Tabel1" style="zoom:80%;" />

**我们提出的BTS架构中，我们主要进行了三步。**

首先，参照现代技术，分析了多种需求，确定了加速器设计时合适的目标和需要。

其次，我们设计了平衡的架构，我们分析了在我们的参数集下同态加密中的计算和合适的数据索引来平衡计算和数据移动。我们还提出了系数级并行取代剩余多项式级并行性避免负载不平衡性问题。

最终，我们提出了现代的PE微架构来有效解决同态计算中不同计算的有效复用性NOC架构。

BTS在乘法上的速度是F1的5714倍，BTS在训练时间中回归的时间相较于CPU快了1306倍，比GPU快了27倍。总体来说比CPU快了5556倍。

**本文的主要贡献：**

提供了同态加密在全同态加密加速器中的性能影响

提出了BTS，一个先进的有大量并行计算单元和FHE定制NOC计算单元的加速计架构

BTS是第一个以实际bootstrapping为目标的加速器

### 2 Background

**此Section主要为介绍CKKS**

我们提供了一个简单的同态加密和CKKS方案，table2是主要参数和本文中的标记。

<img src="C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Table2.png" alt="Table2" style="zoom:80%;" />

#### 2.1 Homomorphic Encryption(HE)

同态加密使得加密数据直接计算，加密的数据称为密文（ct）。有两种同态加密，Leveled HE只支持一定数量的密文计算，因为包含一定的噪声，另一种为全同态加密，支持无限制的密文计算，因为bootstrapping操作可以不断刷新密文中的噪声。

FHE不同方案支持不同明文类型，CKKS方案支持定点复数计算。**本文关注于FHE中的CKKS方案**，同时我们还支持其他常用的FHE方案。

#### 2.2 CKKS: an emerging HE scheme

**CKKS主要流程：**CKKS首先编码一个复数向量的消息变为明文$m(X)=\sum_{i=0}^{N-1}c_iX^i$，这是一个多项式环$R_Q=Z_Q[X]/(X^N+1)$。系数$\{c_i\}$为模数为$Q$的整数，系数的数量为$N$（$N$为2的幂次），通常来说范围为$2^{10}$到$2^{18}$,对于一个$N$，信息至多有$N/2$个复数可以在CKKS中打包为一个单独的明文。每个在打包的信息中的元素叫做$slot$。在编码后在两个消息间的乘法和加法可以通过明文多项式操作实现。$CKKS$然后加密一个明文$m(X)\in R_Q$到一个密文$ct\in R_Q^2$，基于等式$ct=(b(X),a(X))=(a(X)·s(X)+m(X)+e(X),a(X))$，$s(X)\in R_Q$为私钥，$a(X)$为一个随机多项式，$e(X)$为一个小的高斯误差多项式。CKKS解密一个密文$ct$通过计算$m'(X)=ct·(1,-s(X))=m(X)+e(X)$，$m(X)$为含小误差的近似解。

**同态加密的主要瓶颈为多项式高计算复杂度。**因为一个多项式的参数很大（最高可到1000s bits）并且degree很高（位数甚至超过100000），两个多项式间的操作有高计算量和数据传输量。为了解决计算复杂度，HE方案使用了RNS系统，可以将一个大整数划分为多个小数。我们还可以将两个多项式间的计算转化为剩余多项式间字大小的参数计算，来避免大量的算数运算。

#### 2.3 Primitive operations(ops) of CKKS

**HAdd** 对于两个密文$ct_0$，$ct_1$

$ct_{add}=(b_0(X)+b_1(X),a_0(X)+a_1(X))$

**HMult** 包含tensor product和key-switching

tensor product先生成$(d_0(X),d_1(X),d_2(X)):$
$$
\begin{align}
d_0(X)&=b_0(X)·b_1(X)\\
d_1(X)&=a_0(X)·b_1(X)+a_1(X)·b_0(X)\\
d_2(X)&=a_0(X)·a_1(X)\\
\end{align}
$$
key-switch包含一个密钥$evk$

$ct_{mult}=(d_0(X),d_1(X))+P^{-1}(d_2(X)·evk_{mult})$

**HRot** 过程的目的为使得一个消息向量$z=(z_0,...,Z_{N/2-1})$，经过旋转后为$z^{(r)}=(z_r,...,z_{N/2-1},z_0,...,z_{r-1})$

包含一个密钥$evk_{rot}^{(r)}$

$ct_{rot}=(b(X^{5^r}),0)+P^{-1}(a(X^{5^r}))·evk_{rot}^{(r)}$

#### 2.4 Multiplicative level and HE bootstrapping

**Multiplicative level** 

因为加密时误差多项式$e(x)$的存在，过多次的乘法会使得无法复原。于是有**HRescale**的出现，此步将不断舍弃旧模数。乘法level l代表在密文ct上还可以做的乘法次数，因此一个密文ct在level 情况下可以被表示为$N\times(l+1)$的矩阵

**Bootstrapping**

全同态加密中bootstrapping操作可以恢复一个密文的乘法等级level以支持更多的计算。bootstrapping包含同态线性转化和sin近似，可以被分解为多个基础HE操作。bootstrapping操作需要一个$L_{boot}$等级，最大乘法等级L应该大于这个等级，并且更大的最大乘法等级L可以避免过频繁的bootstrapping操作。$L_{boot}$操作大小范围为10到20，这取决于bootstrapping算法，更大的$L_{boot}$可以更精确更快速。我们选择了《Better Bootstrapping for...》中的算法，$L_{boot}$为19，基底$p_i,q_i$同时还应该足够大以容忍bootstrapping积累的误差，通常值在$2^{40}$到$2^{60}$之间。

#### 2.5 Modern algorithmic optimizations in CKKS and amortized mult time per slot

**Security level**

安全等级$\lambda$代表私钥被攻击的复杂度，我们选择的$\lambda$为128bits，$\lambda$ is a strictly increasing function of $𝑁/log_{𝑃Q}
$

**dnum**

在《Better Bootstrapping for...》中，为减少keyswitch中的访存量，讲密文ct分解为$dnum$份，同时辅助密钥$evk$也将变多为$dnum$份、总体来说，计算复杂度会随着$dnum$增长而增长。

**Amortized mult time per slot**
$$
T_{mult,a/slot}=\frac{T_{boot}+\sum_{l=1}^{L-L_{boot}}T_{mult}(l)}{L-L_{boot}}·\frac{2}{N}
$$
$T_{boot}$为bootstrapping的时间，$T_{mult}(l)$为在$l$等级下的乘法时间，$T_{mult,a/slot}$就可以表示$CKKS$的实际运算量。

### 3 Technology-driven parameter selection of bootstrappable accelerators

#### 3.1 Technology trends regarding memory hierarchy

尽管内存带宽的发展，SRAM的内部带宽约为数十TB/s，仍比包含HBM在内的内存带宽高一个数量级。

SRAM容量小，无法存放下全部的辅助密钥$evk$。

接下来讲分析$bootstrapping$的性能影响重要性并提供不同参数如何影响在$bootstrapping$中的数据搬运。

#### 3.2 Interplay between key CKKS parameters

选择一个CKKS中的参数就会对其他参数有多方面的影响。

首先，当$Q$高的时候$\lambda$就会很低，当N很高的时候就会升高。考虑到CKKS需要高的$L$，当模数被设定在$2^{50}$到$2^{60}$作为64位机器字时，$log PQ$最大位500，为了支持128b的安全等级，就必须N大于$2^{14}$

其次，当$log PQ$按照固定的$\lambda$和$N$设定时，更大的dnum将会提高L，同时也会提高辅助密钥evk的大小。考虑到$k=\frac{L+1}{dnum}$，$Q:P\approx dnum:1$。当$logPQ$固定时，更大的dnum意味着更大的Q和更大的L，evk也会变大。当$dnum$增加时L将逐渐饱和（看figure1a图），所以选择合适的dnum很重要。

<img src="C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Figure1.png" alt="Figure1" style="zoom:80%;" />

#### 3.3  Realistic minimum bound of HE accelerator execution time

$T_{mult,a/slot}$主要取决于bootstrapping的时间，因为bootstrapping的时间是一次HMult的60倍时间。

bootstrapping中所需要的evk通常需要40个，主要因为大量的HRots在bootstrapping中使用，就会有很多GB的存储，所以就有很差的本地存储性。

bootstrapping主要时间在HMult和HRot上，而两个操作都是内存密集型，所以可以使用SRAM加速器。

一个evks大小大约会几百MB，但是SRAM无法容纳下。所以受片外带宽限制，加载evk的时间就是最小化实际时间的关键。

#### 3.4 Desirable target CKKS parameters for HE accelerators、

为了理解ckks参数的影响，我们模拟了在去除N、L和dnum时的$T_{mult,a/slot}$

我们在6.2中模拟了bootstrapping，其中共需要19个level，我们增加了两个假设：1，HE计算的时间可以充分被evk访存延迟覆盖；2，所有的密文都在SRAM片中并可以复用。

Figure2中展示了结果，x轴展示了$\lambda$取决于$N\log PQ$，使用的计算工具为[SparseLWE-estimator](https://github.com/Yongyongha/SparseLWE-estimator)。y轴为$T_{mult,a/slot}$。

<img src="C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Figure2.png" alt="Figure2" style="zoom:80%;" />

我们有两个主要发现。

首先，当其他参数固定时，甚至有因为$L-L_{boot}$更大导致ct和evk的增大有着更大的内存压力，随着N增加$T_{mult,a/slot}$反而减小，但是这样的影响在$N=2^{17}$后饱和。在目标的128bit安全等级附近，$2^{16}$到$2^{17}$为3.8倍，$2^{17}$到$2^{18}$为1.3倍。

其次，更大的dnum可以帮助更下的N达到目标的128bit安全等级，但是时间的增加是因为L的增加导致的evk的大小增大。

这些关键的发现指导着HE加速器应该目标于高多项式维度和低dnum值，我们选择的N为$2^{17}$，在图中标记的三组数据中速度分别为$27.7ns,19.9ns,22.1ns$。尽管BTS支持更多的参数，但我们没有进行优化，因为其他的效率会很低或者需要更多的片上资源。

我们在本篇文章中选择的CKKS参数为$N=2^{17},L=27,dnum=1$，密文在最大等级下大小为$56MB$，$evk$大小为$112MB$。

### 4 Architecting BTS

我们探索了BTS的组成，我们的HE加速器架构。我们解决了前期工作的限制，尤其是F1的工作，我们提出了一个适合于支持bootstrapping的CKKS方案的加速器。

Section3.4中假设了HE加速器可以用加载evk的时间隐藏全部的计算时间。

BTS中有大量的并行计算单元，但为了满足最优性要求也不能过多，我们首先分析了HMult和HRot中出现的ketswitching操作，并且有大量的计算和内存需求。

#### 4.1 Computational breakdown of HE ops

![Figure3](C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Figure3.png)

Figure3(a)展示了key-switching的数据流动，Figure3(b)展示了计算复杂度，我们关注于三个部分：$NTT,iNTT,BConv$。

**下方部分内容为介绍两个功能实现原理（已省略部分）**

**Number Theoretic Transform(NTT)**
$$
\begin{align}
&a_1(X)·a_2(X)=iNTT(NTT(a_1(X)))\otimes NTT(a_2(X))\\
&Butterfly_{NTT}(X,Y,W)\rightarrow X'=X+W·Y,Y'=X-W·Y\\
&Butterfly_{NTT}(X,Y,W^{-1})\rightarrow X'=X+Y,Y'=(X-Y)·W^{-1}
\end{align}
$$
**Base Conversion (BConv)**
$$
\underset{C_{\ell} \rightarrow B}{\operatorname{BConv}}\left([a(X)]_{C_{\ell}}\right)=\left \{ \Bigg[\sum_{j=0}^{\ell} \underbrace{\left[[a(X)]_{q_j} \cdot \hat{q}_j{ }^{-1}\right]_{q_j} \cdot \hat{q}_j}_{(1)}\Bigg]_{p_i}\right\}_{0 \leq i<k}
$$


因为BConv不能在多项式执行完NTT运行，iNTT将变回到RNS状态，所以CKKS的总体流程为$iNTT\rightarrow BConv \rightarrow NTT$

#### 4.2 Limitations in prior works and the balanced design of BTS

前几加速器研究研究主要的重点在NTT上，而且F1的研究中N的值为$2^{14}$，如此小的参数，所有的密文和辅助密钥都可以存在片中。

但是我们觉得过多的NTT处理单元过于浪费，因为片外内存带宽已成性能的主要限制条件。而且随着参数的增大，不可能在片中存放所有的数据，并且前期的工作只考虑了最大的dnum情况。

我们分析了NTT处理单元的合适数量。最小NTTU需要的数量为$min_{NTTU}=\frac{\#of\ butterfiles\ per\ HE\ op}{operationg\ frequency}/\frac{size\ of\ an\ evk}{main-memory\ bandwidth}$。我们假设之前工作计算频率为$1.2GHz$，HBM的带宽为1TB/s，那么$min_{NTTU}$如下：
$$
min_{NTTU}=\frac{(dnum+2)·(k+l+1)·\frac{1}{2}NlogN/(1.2GHz)}{2·dnum·(k+l+1)·N·8B/(1TB/s)}
$$
当$dnum=1$时值最大。当$N=2^{17}$时，值为1328，我们在BTS提供了2048个NTTUs已支持其他计算。

总之，对于(i)NTT来说，当dnum变小后，BConv的重要性却更大了。在key-switching中，BConv的计算复杂度为$1+\frac{2}{dnum}$，当$dnum=max$时，BConv的计算复杂度占$12\%$，但是当$dnum=1$时，却占用$34\%$（figure3(b)），我们提出要使用先进的BConv处理单元来解决BConv重要性的提高，具体在Section5.2。

#### 4.3 BTS organization exploiting data parallelism

![Figure4](C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Figure4.png)

我们可以根据数据访问模式分类基础HE操作为三类。第一种为剩余多项式方向，(i)NTT和自同构的操作、第二种为系数方向，BConv包含了所有(l+1)个剩余环的单独系数产生一个输出。第三种为单基础部分，如CMult和PMult。

我们可以产生两种数据并行，剩余多项式级并行(rPLP)和系数级并行(CLP)。剩余多项式级可以分为(l+1)部分和系数级并行分为N个系数到许多的处理单元实现并行。

当数据模数和并行方式没有确定，处理单元间数据的交换就会出现，最终就会导致总线间的数据交换。在$key-switching$中的$iNTT\rightarrow BConv \rightarrow NTT$，CLP将会出现在(i)NTT，rPLP将会出现在BConv中。两种方式的数据传输量均为$(k+l+1)N$，所以在数据交换上没有更好的并行方式。但是由于l的变化，rPLP的并行会受到限制，而且会使得处理单元间的数据分配变得复杂。

我们最终采用了CLP在BTS中。因为N是一个比较固定的值，所以我们选择一个固定数据分配的方式，剩余多项式同一系数地址的数据将被分配在同一个处理单元中。所以只有(i)NTT和自同构的程序需要处理器外部间的数据交换。

我们设置了2048个处理单元，每个处理单元包含一个NTT处理单元，一个BConv处理单元，一个ModAdd处理单元，一个ModMult处理单元。 $N=2^{17}$时包含了同量的剩余多项式环，所以每个处理单元处理$2^6$个剩余环。那么17个中的6个(i)NTT可以单独在处理单元中计算（？）。我们采用了3D-NTT来最小化处理单元中的数据交换。一个剩余多项式倍看作一个$2^6\times 2^5\times 2^6$的3D数据结构。A residue polynomial is regarded as a 3D data structure of size $2^6 × 2^5 × 2^6$ . Then, each PE performs a sequence of 26 -, 25 -, and 26 -point (i)NTTs, interleaved with just two rounds of inter-PE data exchange. Splitting (i)NTT in a more fine-grained manner requires more data exchange rounds and is thus less energy-efficient. The automorphism function exhibits a different communication pattern from (i)NTT, involving complex data remapping (Eq. 5). Nevertheless, the data distribution methodology and NoC structure of BTS efficiently handle data exchanges for both (i)NTT and the automorphism (see Section 5).

（这段没看懂）

### BTS Microarchitecture

![Figure5](C:\Users\cznem\Desktop\华为云盘\学习资料\科创\Paper\BTS An Accelerator for Bootstrappable Fully Homomorphic Encryption\Figure5.png)

我们设计了一个大规模并行架构来网格状分发处理单元。一个处理单元由fuction处理单元和一个SRAM存储器构成。一个NTT处理单元在每个处理单元(PE)中(i)NTT阶段处理剩余多项式中的一部分。因为使用了CLP的数据分割方式，另外两种类程序可以在同一PE中计算而不需要数据交换。

Figure5展示了高维的BTS。我们使用了2048个PE在一个网格中，高度为32，宽度为64。PE间连接通过按维度排列的crossbar，有$32\times 32$横向crossbar和$64\times 64$纵向crossbar。我们使用了一个中心恒定的内存来存储预处理的参数，broadcast unit传输预处理的值到PE中。内存控制器在顶部和底部，每个连接一个HBM栈。BTS接受指令和必要的数据通过PCIe接口。BTS中的字长为64bits。模降算法使用Barrett reduction来将128位结果重回至字长。

#### 5.1 Datapath for (i)NTT

。。。

#### 5.2 Base Conversion Unit(BConvU)

BConv由两部分组成，第一部分是乘$\left[\hat{q}_j^{-1}\right]_{q_j}$,第二部分是乘$\left[\hat{q}_j\right]_{p_i}$然后求和。

一个BConv处理单元有一个ModMult处理单元来计算第一部分，有一个乘并求和的处理单元来计算第二部分。

因为第二部分的计算依赖iNTT的完成，所以必须等待iNTT的完成，我们调整了BConv的式子。
$$
\left\{\sum_{j_1=0}^{(\ell+1) / l_{\text {sub }}}\left[\sum_{j_2=j_1 \times l_{\text {sub }}}^{\left(j_1+1\right) \times l_{\text {sub }}-1}\left[[a(X)]_{j_2} \cdot \hat{q}_{j_2}^{-1}\right]_{q_{j_2}} \cdot \hat{q}_{j_2}\right]_{p_i}\right\}_{0 \leq i<k}
$$
这样使得第二部分在iNTT进行过程中就可以开始，第一部分在$l_{sub}$（bts中等于4）完成并存储在$RF_{MMAU}$。$MMAU$计算对应部分和，然后将前段结果求和，这部分将存储和加载在SRAM存储器(Scratchpad)内以减少每轮次的读写。临时计数器和FIFO最小化了$RF_{MMAU}$的带宽压力并正确的转移了数据。预处理的数据$\left[\hat{q}_j^{-1}\right]_{q_j}$和$\left[\hat{q}_j\right]_{p_i}$将在需要的时候加载到$RF_{BT1}$和$RF_{BT2}$。

我们还分析了MMAU对其他操作的影响力。在keyswitching最后还包含了减法、放缩和加法，表达为：$\left[d 2^{\prime} . a x\right]_{Q_{\ell}} \times(1 / P)+\left[d 2^{\prime} . a x\right]_{P \rightarrow Q_{\ell}} \times(-1 / P)+ d 1 \times 1+0 \times 0$，因此，我们融合了这三个操作，可以在MMAU上计算，我们把这个融合体叫做subtraction-scaling-addion(SSA)。

#### 5.3 Scratchpad

每个PE中的Scratchpad有三个功能。

第一，存储HE计算中的临时数据。keyswitching中的临时数据很大，可能到达28MB，如果不在片中会非常影响性能。

第二，存储预存取的辅助密钥evk来减少加载的延迟。

第三，作为密文ct的cache，受软件控制。ct有很高的数据复用性。

在BConvU中Scratchpad的带宽需求很高，因为其中包含的求和。在前面的等式中需要加载$(l+1)/l_{sub}$次，带宽压力可以通过提高$l_{sub}$减小，但是这可能需要MMAU中更多的线路。

#### 5.4 Network-in-Chip(NoC) design

BTS中有三种交流方式：1，片外内存到PE（PE-Mem NoC）；2，预计算数据到PE（BrU NoC）；3，PE间数据交换包含(i)NTT和自同构（PE-PE NoC）。BTS的节点很多并需要很高的带宽。BTS中提供了三种独立的NoC而不是一条公用的NoC。

**PE-Mem NoC:** 将片内的PE分为32个区域，因为每个HBM2e支持16通道，并且片内有2个HBM控制器，所以每个区域链接一个HBM通道，每个区域包含了64个PE

**BrU NoC:** 片中包含128个BrUs，每个为16个PE提供数据。

**PE-PE NoC:** 通信方式和内容已知，x轴方向使用xbar通信，y轴方向使用ybar通信。

#### 5.5 Automorphism

### 6 Evaluation

...
