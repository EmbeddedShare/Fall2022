## A Full RNS Variant of Approximate Homomorphic Encryption

Author:  Jung Hee Cheon, Kyoohyung Han, Andrey Kim, Miran Kim  and Yongsoo Song

research unit: Seoul National Univeristy, University of California

### Abstract

$HE$(同态加密)近年来发展很快，有很多新的技术，2017年提出的$CKKS$加密方案有着对于real number加密的最好方案，但是没有使用$RNS$(Residue Number System)和$NTT$(Number Theoretic Transformation)技术。

在本篇文章中，我们使用了$RNS$和$NTT$技术，相比于别的做法，我们能支持在64位内实现运算。

通过实验，我们的方案速度很快，在解密，普通乘法和同态乘法分别快$17.3,6.4,8.3$倍。

### 1 Introduction

随着大数据分析的发展和数据安全的需要，HE的需要越来越广泛。

现在有很多软件的$HE$方案都是基于$RLWE$问题(Ring Learning with Errors)，如$HElib$的$BGV$和$SEAL$的$BFV$。这些方案都是基于分圆多项式的剩余环，导致了计算性能的逐步衰弱。Gentry等人的研究[22]对于一种多项式计算的执行方案提出了一种分圆多项式的代表，叫做$double-CRT$，基于中国剩余定理($CRT$)。首层$CRT$使用了$RNS$来分解一个多项式成一组有小模数的多项式。在第二层中通过$NTT$转换每个小多项式为一个有小模数的向量。在$double-CRT$当中，一个任意的多项式都可以表示成一个**由小数构成的矩阵**，使得多项式能通过离散模操作进行算术。

最近，有$HEAAN$的研究($CKKS$)支持了近似数算法。

但是在原有的$CKKS$算法中有一个重要的问题就是$double-CRT$的使用。$HEAAN$的rounding操作可以通过两个连续密文模的比值分为加密明文，所以说一个密文的模应该使用2的幂次或者质数，但是这对于$RNS$是不友善的，所以说之前的方案在相同的参数情况下比别的HE方案慢。

#### Our Contribution

在本文中，我们了一种基于分圆多项式环的$double-CRT$的$HEAAN$方案。主要的idea就是利用一组由固定基的近似值组成的基作为我们的模数。在$HEAAN$中每个信息都有误差，但是在我们的忍受范围之内。总之我们集成了$double-CRT$和之前的方案。

同时我们还介绍了一些$HEAAN$中不需要$RNS$的计算。我们新的模数切换(modulus switching)可以替换之前方案中无算术操作。

我们的方案相比于之前提升很大，基础操作可以提升10倍速度。

最终，我们提出了一种梯度下降算法来展示我们的HE库的在真实世界中复杂运算。在一个数据集里，之前的方案在**4核**处理器下运行**3.5分钟**，而我们的方案在**单核**下运行**1.8**分钟。

#### Technical Details

1.

$N$等于一个2的幂次的整数

$R=Z[x]/(X^N+1)$代表第$2N$个的分圆多项式环

对于一个固定的基$q$，我们选择一个$RNS$基$\{q_0,...,q_L\}$，是与基$q$近似同样$size$的互质的数。

对于一个整数$0\leq l\leq L$，一个在$l$层的密文可以经过$rescaling$操作转换到$l-1$层。

原方案比较灵活，因为对于一个密文来说可以使用任何一个数组成起来进行$rescaling$，相比于RNS中的基$q$更灵活，但是我们的方案中可以在性能上明显提升。

2.

我们的方案中支持$NTT$和$RNS$多项式分解。

$NTT$在基$q_l$是质数时满足$q_l \equiv 1 (mod 2N)$时效率较高。

我们给出了一个list，包含了一些不同的质数同时满足了$double-CRT$中两层(NTT,RNS)的情况。

3.

$HEAAN$算法中包含了将$R_Q$转换到$R_{P·Q}$，$P$是一个大整数，来让模数扩大。这些无运算的操作在$RNS$上操作很困难，为了最优化，我们采用了[3]中的想法，用近似的模数来转换，同时有小误差。

我们没有在原有方案中精确的计算，而我们的近似模数提出的算法要寻找一个$\tilde{a}\in R_{P·Q}$满足$\tilde{a}\equiv a(mod\ Q),||\tilde{a}||\ll P·Q$，对于多项式$a\in R_Q$，反过来说，近乎模数减少算法返回了一个对象$b\in R_Q$，如$P·b\approx\tilde{b}$，对于一个输入的多项式$\tilde{b}\in P_{P·Q}$，这些过程在输出的多项式上给出了宽松的情况。总之，我们表明了HE系统在有小的新增错误时的正确性。

#### Related Works

有些关于HE的研究不支持$rounding\ operation$，这使得随着乘法深度的加深，密文模数的位数将成指数级增长。

很多的HE方案使用了有大系数的多项式环结构，最近的研究也加速了使用RNS加速昂贵的环操作。Bajard等人的[3]提出了一个完全$RNS$的BFV方案，他们的执行中可以避免在同态计算时避免RNS和环元素的系数的转换。在这之后，Halevi等人的[25]提出了一个朴素的不会产生更多噪声的方法，在这个idea之上，可以在执行中让HE方案中不需要任何大数计算的库，这个已经被更新到最近的SEAL库中。

#### Road-map

Section 2：$HEAAN$的基础和快速基变换的介绍

Section 3：我们展示了一个使用$RNS$加速全面同态运算的方法

Section 4：我们描述了一个完整$RNS$类的$HEAAN$

Section 5：最优化技术的实验结果

### 2 Background

向量：加粗字体

$<·,·>$：点积

对于实数$r$，$\lfloor r\rceil$代表最接近的整数

对于一个整数$q$，$Z\cap (-q/2,q/2)$表示为$Z_q$，$[a]_q$表示a模q

$x\leftarrow D$表示从分布$D$中取出$x$

$U(S)$表示在$S$是有限集时$S$的均匀分布。

$\lambda$表示安全系数

$B=\{p_0,p_1,...,p_{k-1}\}$表示一组基，其中都是成对互质的

#### 2.1 Approximate Homomorphic Encryption

定义：

对于一个2的幂次的整数$N$,表示第$2N$个分圆域$K=Q[x]/(X^N+1)$，$R=Z[X]/(X^N+1)$为整数环。剩余环模一个整数$q$表示为$R_q=R/qR$。

HEAAN方案使用了一个固定的整数基$q$，然后构造了一条模链$Q_l=Q^l,1\leq l\leq L$。

对于一个多项式$m(x)\in K$，一个密文$ct$在$ct\in R^2_{Q_{l}},[<ct,sk>]_{Q_l}\approx m(x)$情况下被称为$m(X)$加密的$l$等级。

同态运算 ：

HEAAN中在密文间同态运算可以被使用特殊模数的$key-swiching$完成，在[22]中有介绍。对于一个加密的输入$m_1(X),m_2(X)$处在$l$等级，他们的加和乘是$[<ct_{add},sk>]_{Q_l}\approx m_1(X)+m_2(X)$，$[<ct_{mult},sk>]_{Q_l}\approx m_1(x)·m_2(x)$

重放缩：

这个方案的主要优势是内在的叫做重放缩的运算(rescaling)。重放缩算法表示为$RS(·)$，可以把一个$l$级的$m(X)$加密后的密文转化到$l-1$级的$q^{-1}·m(X)$加密后的密文，这个可以被当作近似rounding运算或者提炼最重要的位数出来。通过减小密文的大小，我们可以减少在运算中的模数速度消耗

编码和解码：

为了压缩乘法的信息，有一个方法定义为一个通过一种经典嵌入的有复数向量的分圆域。让$\zeta=exp(-\pi i/N)$为第$2N$个在复数集(C)上的原根。传统的嵌入的$K$被定义为$a(X)\rightarrow(a(\zeta),a(\zeta^3),...,a(\zeta^{2N-1}))$。发现没有需要存储所有$\sigma(a)$来提取$a(X)$，因为$a(\zeta^j)=\overline{a(\zeta^{2N-j})}$，我们表示$\tau:K\rightarrow C^{N/2}$一种传统嵌入，定义为：$\tau:a(X)\rightarrow (a(\zeta),a(\zeta^5),...,a(\zeta^{2N-3}))_{0\leq j<N/2}$。然后使用这个来在HEAAN中解码。$\tau$的反函数被使用在编码中压缩$(N/2)$个辅助在一个多项式中。

加密和解密：

这个方案可以被使用在实数或者复数上，我们乘一个放缩因子$q$到一个有限精度的数$z$，然后我们使用$m=q·z$来解密。同时加密的数满足$[<ct,sk>]_{Q_l}\approx q·z$，两个加密的密文$q·z_1,q·z_2$将会被返回一个加密的$q^2·z_1z_2$，然后放缩因子变为了$q^2$，通过重放缩可以将放缩因子重新变为$q$

#### 2.2 The RNS Representation

让$B=\{p_0,...,p_{k-1}\}$作为基，$P=\prod ^ {k-1}_{i=0}p_i$。我们表示$[·]_B$代表$Z_p$到$\prod ^ {k-1}_{i=0}Z_{p_i}$的索引，定义为$a\rightarrow [a]_B=([a]_{p_i})_{0\leq i <k}$。这是从CRT来的环同构同时$[a]_B$被叫做$RNS$的代表$a\in Z_P$。$RNS$的主要优点是可以执行内部组件多的算术运算到一个小的环$Z_P$，可以减少近似和实际的运算消耗。这个在整数上的环同构可以被自然的拓展到一个环同构$[·]_B:R_P\rightarrow R_{p_0}\times ... \times R_{p_{k-1}}$通过使用一个丰富西数的分圆环。

#### 2.3 Fast Basis Conversion

Brakerski[6]介绍了一个尺度不变的基于$LWE$问题的HE方案，Fan和Vercauteren[19]提出了基于环的方案叫做$BFV$。最近$Barjard$[3]等人提出了一种包含在同态计算时RNS表现密文的BFV方案。这个方案提出了一个新的算法，叫做$Fast\ Basis\ Conversion$（FBC，快速基变换），来转换剩余的多项式到一个与之前基同余的新基。

更精确的来说，对于一个基$\{p_0,...,p_{k-1},q_0,...,q_{l-1}\}$，让$B=\{p_0,...,p_{k-1}\},C=\{q_0,...,q_{l-1}\}$作为子集，我们表示他们的乘积$P=\prod ^ {k-1}_{i=0}p_i,Q=\prod ^ {l-1}_{j=0}q_j$。一个可以进行RNS的转换的$[a]_c=(a^{(0)},...,a^{(l-1)})\in Z_{q_0}\times ...\times Z_{q_{l-1}}$中的元素$a\in Z_Q$到一个$Z_{p_0}\times ...\times Z_{p_{k-1}}$的元素，计算方式为：
$$
Conv_{c\rightarrow B}([a]_C)=(\sum_{j=0}^{l-1}[a^{(j)}·\hat{q_j}^{-1}]_{q_j}\ \ (mod \ p_i))_{0\leq i<k}
$$
其中$\hat{q_j}=\prod _{j'\neq j}q_{j'}\in Z$，我们注意到对于一个小的数$e\in Z$满足$|a+Q·e|\leq (l/2)·Q$，$\sum_{j=0}^{l-1}[a^{(j)}·\hat{q_j}^{-1}]_{q_j}·\hat{q_j}=a+Q·e$。这意味着$Conv_{c\rightarrow B}([a]_c)=[a+Q·e]_B$可以被看做$a+Q·e$在基$B$下的$RNS$转换。

### 3 Approximate Bases and Full RNS Modulus Switching

$CKKS$中在近似数上的算术有自己的有点，但是一个密文模数不能被选座乘互质数的乘积，所以这个操作[11]需要昂贵的乘精度模数学。在这个section，我们介绍了一种可以避免使用2的幂次模密文模然后允许在$HEAAN$中进行$RNS$分解，我们也提出了新的算法来转换密文模数到$RNS$组件上。

#### 3.1 Approximate Basis

在$CKKS$中的rescaling中，基$Q_l=q^l$，这个在使用RNS表示上很困难。

为了解决这个问题，我们提出了一种新的想法来使用近似值来组成RNS的基。更细节的来说，给出一个放缩因子$q$和位精度$\eta$，我们寻找一个基$C=\{q_0,...,q_L\}$如对于$l=1,...,L,s.t.\ q/q_l\in (1-2^{-\eta},1+2^{-\eta})$。

不影响精度的证明：在$l$级的密文模数为$Q_l=\prod_{i=0}^lq_i$，所以相邻两项的比为$Q_l/Q_{l-1}=q_l\approx q$，所以可以将$l$级的密文转换为$q_l^{-1}·m$的$l-1$级密文，两项的误差为$|q_l^{-1}·m-q^{-1}·m|=|1-q_l^{-1}·q|·|q^{-1}·m|\leq 2^{-\eta}·|q^{-1}·m|$，是在精度要求的$\eta$位之内的。

#### 3.2 Approximate Modulus Switching	

$CKKS$中有非算术的运算，这些运算不能直接执行到$RNS$组件，特别的来说，同态乘法和重线性化过程需要一个精选的模数切换算法，同时key-switching技术对于旋转和共轭也包含了同样的运算。

我们评论了模数切换算法的目的在[12]中，可以被减少到一个寻找在保证HE方案正确性时有小错误的密文的问题，在这个section中，我们提出一个模糊化运行模数切换算法到RNS表示上的idea，一个Full variant的$HEAAN$将会在下一个section中介绍，同时基于这个方法。通过这个paper，我们将表示$D=\{p_0,...p_{k-1},q_0,...,q_{l-1}\},B=\{p_0,...,p_{k-1}\},C=\{q_0,...,q_{l-1}\}$作为基和子集，分别的积为$P=\prod_{i=0}^{k-1}p_i,Q=\prod_{j=0}^{l-1}q_j$。

##### Approximate Modulus Raising

假设我们有RNS表示的$[a]_c,a\in Z_Q$，表示为$ModUp$的近乎模数升高算法的目的是发现一个数$\tilde{a}\in Z_{PQ}$关于基D的RNS表示，同时满足$\tilde{a}\equiv a\ (mod\ Q),|\tilde{a}|\ll P·Q$。在$[\tilde{a}]_C=[a]_C$的条件下，我们只需要生成在B基的$\tilde{a}$的$RNS$表示，同时这个可以被快速变换算法实现，Algorithm 1是近似模数升高的伪代码

**Algorithm 1** Approximate Modulus Raising
**procedure** $ModUp_{c\rightarrow D}(a^{(0)},a^{(1)},...,a^{(l-1)})$

$(\tilde{a}^{(0)},...,\tilde{a}^{(k-1)})\leftarrow Conv_{c\rightarrow B}([a]_C).$

**return ** $(\tilde{a}^{(0)},...,\tilde{a}^{(k-1)},a^{(0)},...,a^{(l-1)})$

**end procedure**

如Section 2.3说所说，在Algorithm 1里的快速变换算法返回了$[a+Q·e]_B\in \prod_{i=0}^{k-1}Z_{p_i},|e|\leq l/2$。因此$ModUp$算法的输出就是RNS表示的$\tilde{a}=a+Q·e$在$D=B\cup C$的基下。

##### Approximate Modulus Reduction

模数下降算法表示为$ModDown$。输入为$[\tilde{b}]_D,\tilde{b}\in Z_{P·Q}$，计算目标是$[b]_C,b\in Z_Q$，满足$b\approx P^{-1}·\tilde{b}$

近乎模数下降的目的是找到一个小的$\tilde{a}=\tilde{b}-P·b$，满足$\tilde{a}\equiv \tilde{b}\ (mod\ P)$，RNS表示的$[\tilde{b}]_D$是$[\tilde{b}]_B,[\tilde{b}]_C$的连结。我们首先去第一个部分$[\tilde{b}]_B=(\tilde{b}^{(0)},...,\tilde{b}^{(k-1)})$，就和$[a]_B,a=[\tilde{a}]_P\in Z_P$一样，然后我们使用快速变换算法来计算$RNS$表示$[\tilde{a}]_C,\tilde{a}=a+P·e$，对于一个小的数$e$。注意到$\tilde{a}\equiv \tilde{b}\ (mod\ P),|\tilde{a}|\ll P·Q$，来自于$Conv_{b\rightarrow C}(·)$的性质。最终我们提取$RNS$表示$b=P^{-1}·(\tilde{b}-\tilde{a})$在基$C$下，计算$(\prod_{i=0}^{k-1}p_i)^{-1}·([\tilde{b}]_C-[\tilde{a}]_C)\in \prod_{j=0}^{l-1}Z_{q_j}$，algorithm 2为伪代码：

**Algorithm 2** Approximate Modulus Reduction

**procedure** $ModDown_{D\rightarrow C}(\tilde{b}^{(0)},\tilde{b}^{(1)},...,\tilde{b}^{(k+l-1)})$

​	$(\tilde{a}^{(0)},...,\tilde{a}^{(l-1)})\leftarrow Conv_{b\rightarrow C}(\tilde{b}^{(0)},...,\tilde{b}^{(k-1)})$

​	**for** $0\leq j <l$ **do**

​		$b^{(j)}=(\prod_{i=0}^{k-1}p_i)^{-1}·(\tilde{b}^{(k+j)}-\tilde{a}^{(j)})(mod\ q_j)$

​	**end for**

**return** $(\tilde{b}^{(0)},...,\tilde{b}^{(l-1)})$

**end procedure**

##### Word Operations

paper中的算术操作（如加法和乘法）模一个"word-size"整数叫做word operations。

$Conv_{C\rightarrow B}([a]_c)$输出的$(\sum_{j=0}^{l-1}[a^{(j)}·\hat{q_j}^{-1}]_{q_j}\ \ (mod \ p_i))_{0\leq i<k},\hat{q_j}=\prod _{j'\neq j}q_{j'}\in Z$，可以避免计算大数$\hat{q_j}$，总之，如果我们提前计算和存储这些值，只取决于基$B$和$C$，这样$Conv_{C\rightarrow B}(·)$算法可以减小到$O(k·l)$word operations。

##### Complexity of Approximate Modulus Switching

我们模数z转换的算法有一个优点，就是都可以只使用word operations计算。

近似模数转换算法可以拓展到多项式环上的算法：
$$
\begin{align}
ModUp_{C\rightarrow C}(·)&:\prod_{j=0}^{l-1}R_{p_j}\rightarrow \prod_{i=0}^{k-1}R_{p_i}\times\prod_{j=0}^{l-1}R_{q_j}\\
ModDown_{D\rightarrow C}(·)&:\prod_{i=0}^{k-1}R_{p_i}\times\prod_{j=0}^{l-1}R_{q_j}\rightarrow \prod_{j=0}^{l-1}R_{p_j}
\end{align}
$$
当$N$是一个2的幂次分圆环的度，这些操作需要$O(k·l·N)$word operations。

### 4 A Full RNS Variant of the Approximate HE

$N$：一个2的幂次的整数

$K=Q[x]/(X^N+1)$：第$2N$个分圆域

$R=Z[x]/(X^N+1)$：K上的整数环

一个任意的元素K可以表现成一个有有理数系数的度严格小于N的多项式，可以被识别成系数是$Q^N$的向量，在K上的rounding操作，和模数操作都会被定义为系数丰富rounding和模操作。

**Setup(q,L,**$\eta,1^\lambda$**)**:一个基整数$q$，等级$L$，位精度$\eta$，安全指数$\lambda$

·选择一个基$D=\{p_0,...,p_{k-1},q_0,q_1,...,q_L\},q_j/q\in(1-2^{-\eta},1+2^{-\eta})$，$B=\{p_0,...,p_{k-1}\},C=\{q_0,..,q_l\},D_l=B\cup C_l,P=\prod_{i=0}^{k-1}p_i,Q=\prod_{j=0}^{L}q_j$

·选择一个二的幂次整数$N$

·选择一个私钥$\chi_{key}$，一个加密$\chi_{enc}$，和一个误差$\chi_{err}$超过$R$

·让$\hat{p_i}=\prod_{0\leq i'<k,i'\ne i}p_{i'},0\leq i <k$，计算$[\hat{p_i}]_{q_j},[\hat{p_i}^{-1}]_{p_i},0\leq i<k,0\leq j \leq L$

·计算$[p^{-1}]_{q_j}=(\prod_{i=0}^{k-1}p_i)^{-1}(mod\ q_j),0\leq j\leq L$

·让$\hat{q_{l,j}}=\prod_{0\leq j' \leq l,j'\ne j}q_{j'},0\leq j\leq l\leq L$，计算$[\hat{q_{l,j}}]_{p_i},[\hat{q_{l,j}}^{-1}]_{q_j},0\leq i<k,0\leq j\leq l\leq L$。

**KSGen(**$s_1,s_2$**)**

对于给出的私密的多项式$s_1,s_2\in R$，选取一致的元素$(a'^{(0)},...,a'^{(k+L)})\leftarrow U(\prod_{i=0}^{k-1}R_{p_i}\times\prod_{j=0}^{L}R_{q_j})$和一个误差$e'\leftarrow \chi_{err}$，输出的$switching \ key,swk$为$(swk^{(0)}=(b'^{(0)},a'^{(0)}),...,swk^{(k+L)}=(b'^{(k+L)},a'^{(k+L)})\in \prod_{i=0}^{k-1}R_{p_i}^2\times \prod_{j=0}^{L}R_{q_j}^{2}$，$b'^{(i)}\leftarrow -a'^{(i)}·s_2+e'(mod\ p_i),0\leq i <k,b'^{(k+j)}\leftarrow -a'^{(k+j)}·s_2+[P]_{q_j}·s_1+e'(mod\ q_j),0\leq j \leq L$

**KeyGen**

·选取$s\leftarrow \chi_{key}$，私钥为$sk\leftarrow (1,s)$

·选取$(a^{(0)},...,a^{(L)})\leftarrow U(\prod_{j=0}^{L}R_{q_j}),e\leftarrow \chi_{err}$，公钥为$pk\leftarrow (pk^{j}=(b^{(j)},a^{(j)})\in R_{q_j}^2)_{0\leq j \leq L}$,$b^{(j)}\leftarrow -a^{(j)}·s+e (mod \ q_j),0\leq j \leq L$

·辅助计算密钥$evk\leftarrow KSGen(s^2,s)$

**En**$c_{pk}$**(m)**

对于$m\in R$，选取$v\leftarrow \chi_{enc},e_0,e_1\leftarrow \chi_{err}$，输出的密文$ct=(ct^{(j)})_{0\leq j \leq L}\in \prod_{j=0}^LR_{q_j}^2,ct^{(j)}\leftarrow v·pk^{j}+(m+e_0,e_1)(mod\ q_j),0\leq j \leq L$

**De**$c_{pk}$**(ct)**

对于密文$ct=(ct^{(j)})_{0\leq j \leq l}$，输出$<ct^{(0)},sk>(mod\ q_0)$

$Add(ct,ct')$

给出两个密文$ct=(ct^{(0)},...,ct^{(l)}),ct'=(ct'^{(0)},...,ct^{(l)})\in\prod_{j=0}^lR_{q_j}^2$，输出的密文$ct_{add}=(ct_{add}^{(j)})_{0\leq j\leq l},ct^{(j)}\leftarrow ct^{(j)}+ct'^{(j)}(mod\ q_j),0\leq j \leq l$

$Mult_{evk}(ct,ct')$

给出两个密文$ct=(ct^{(0)},...,ct^{(l)})_{0\leq j \leq l},ct'=(ct'^{(0)},...,ct^{(l)})$，计算结果密文$ct_{mult}\in\prod_{j=0}^jR_{q_j}^2$

1.对于$0\leq j\leq l$计算
$$
\begin{align}
d_0^{(j)}&\leftarrow c_0^{(j)}c_0'^{(j)}\ (mod\ q_j)\\
d_1^{(j)}&\leftarrow c_0^{(j)}c_1'^{(j)}+c_1^{(j)}c_0'^{(j)}\ (mod\ q_j)\\
d_2^{(j)}&\leftarrow c_1^{(j)}c_1'^{(j)}\ (mod\ q_j)\\
\end{align}
$$
2.计算 $ModUp_{C_l\rightarrow D_l}(d_2^{(0)},...,d_2^{(l)})=(\tilde{d_2}^{(0)},...,\tilde{d_2}^{(k-1)},d_2^{(0)},...,d_2^{(l)})$

3.计算
$$
\tilde{ct}=(\tilde{ct}^{(0)}=(\tilde{c_0}^{(0)},\tilde{c_1}^{(0)}),...,\tilde{ct}^{(k+j)}=(\tilde{c_0}^{(k+l)},\tilde{c_1}^{(k+l)}))\in \prod_{i=0}^{k-1}R_{p_i}^2\times\prod_{j=0}^tR_{q_j}^2
$$
其中$\tilde{ct}^{(i)}=\tilde{d_2}^{(i)}·evk^{(i)}(mod\ p_i),\tilde{ct}^{(k+j)}=d_2^{(j)}·evk^{(k+j)}(mod\ q_j)$,$0\leq i <k,0\leq j \leq l$

4.计算
$$
\begin{align}
(\tilde{c_0}^{(0)},...,\tilde{c_0}^{(l)})&\leftarrow ModDown_{D_l\rightarrow C_l}(\tilde{c_0}^{(0)},...,\tilde{c_0}^{(k+l)}),\\
(\tilde{c_1}^{(0)},...,\tilde{c_1}^{(l)})&\leftarrow ModDown_{D_l\rightarrow C_l}(\tilde{c_1}^{(0)},...,\tilde{c_1}^{(k+l)}).
\end{align}
$$
5.输出 密文$ct_{mult}=(ct_{mult}^{(j)})_{0\leq j \leq l},ct_{mult}^{(j)}\leftarrow (\tilde{c_0}^{(j)}+d_0^{(j)},\tilde{c_1}^{(j)}+d_1^{(j)})(mod\ q_j),0\leq j \leq l$

**RS(ct)**

对于一个$l$等级的密文$ct=(ct^{(j)}=(c_0^{(j)},c_1^{j}))_{0\leq j\leq l}\in \prod_{j=0}^{l}R_{q_j}^{2}$，计算$c_i'^{(j)}\leftarrow q_l^{-1}·(c_i^{(j)}-c_i^{(l)})(mod\ q_j),i=0,1,0\leq j< l$,输出密文为$ct'\leftarrow(ct'^{(j)}=(c_0'(j),c_1'^{j}))_{0\leq j\leq l-1}\in \prod _{j=0}^{l-1}R_{q_j}^2$

### 5 Software Implementation

开源网址：[https://github.com/HanKyoohyund/FullRNS-HEAAN](https://github.com/HanKyoohyund/FullRNS-HEAAN)

结果是，同态运算的复杂度主要取决于两个NTT表现的转换
