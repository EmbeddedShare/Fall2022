## Insider: Designing In-Storage Computing System for Emerging High-Performance Drive

这篇是加州大学洛杉矶分校的一篇ISC工作，相对于其他ISC工作其创新在于

(1)提高ISC的灵活性，支持多种应用场景下的ISC任务；

(2)提供完成的系统抽象和编程模型，在Host端使用类似文件操作的API，Drive端的加速程序只考虑计算逻辑，而不用维护复杂的数据移动逻辑，在Host端和Drive开发成本和难度都大大缩小；

(3)支持并发处理多项ISC任务，并解决并发情况下数据保护、进程隔离性和带宽调度等问题；

### Motivation

随着存储技术的发展，大数据分析场景下的系统瓶颈从存储驱动转移到host/drive之间的数据通路和host的IO栈上，因此相比与将数据搬运到host处理，现有的工作探究将计算任务从host下放到drive上。现有的基于嵌入式ARM核驱动和ASIC方式存在如下问题：

1. 有限的性能和灵活性。drive中的嵌入式核计算能力太弱，ASIC只适合某种专用负载。
2.  大量的编程工作代价。host端现有系统为ISC设计的接口没有适配性，因此需要在host端修改大量代码。drive端程序需要维护host和drive之间的元素据。
3. 缺乏系统支持。现有工作没有考虑多个进程共享drive的情况，避免下方的读写未授权的数据的数据保护也没有实现。

为解决上述问题，INSIDER完成以下任务：

1. 饱和高驱动速率。INSIDER使用FPGA作为ISC单元，数据减少等遗留代码提取到drive程序中，采用系统级的流水线。

2. 提供高效的抽象。在host端INSIDER将ISC抽象为文件操作，隐藏ISC单元；在drive端，将下放的任务提供一个仅计算的抽象。

3. 提供必要的系统支持。INSIDER分为控制平面和数据平面，控制平面不可编程，负责动态发起drive access请求、安全检查等，数据平面的ISC单元在drive的DMA单元和存储设备之间处理数据。此外INSIDER配备自适应驱动带宽调度器，控制平面根据该调度器动态发起请求。
4. 更高的成本效益。定义为每美元的有效数据处理率。

### Background

1. 新兴存储设备

   如Figure3a所示的传统数据处理架构的设计依赖于2个假设：1. interconnection速度高于drive速度；2host端的IO stack执行速度高于drive速度。该假设已经不适用现有的存储设备，如Figure1和Figure2所示，storage drive的速度已有显著提升，但interconnection bus停滞不前。

   ![image-20230109211603842](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301092116911.png)

![image-20230109211620472](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301092116514.png)

最近的平台大多采用PICe gen3，每个连接可以达到1GB/s的双向通信速度。但现有16通道，单个bank的存储设备即可达到6.4GB/s，越来越大的内外部带宽差异使得不能充分发挥drive的性能。host IO栈的延迟也远大于PCIe往返延迟和SSD的读写延迟。总结下就是一个“数据移动瓶颈”的问题

2. ISC

如Figure3b所示，为解决上述问题提出ISC架构，通过将部分工作下放到ISC accelerator，较少drive到host数据量，缓解interconnection和host IO stack的瓶颈，自定义的IO stack绕过传统OS stack达到更低的延迟。随着VLSI技术的发展，将计算资源集成到存储设备上的成本大大降低，存储设备的性能也大大提升，超过Interconnectin，驱动着ISC技术的发展。现有的基于嵌入式ARM核驱动和ASIC方式存在如下问题：

1. 有限的性能和灵活性。drive中的嵌入式核计算能力太弱，ASIC只适合某种专用负载。
2.  大量的编程工作代价。没有为程序员提供高效的编程抽象。host端为ISC设计的接口可能不适配现有系统如POSIX，因此需要在host端修改大量代码。drive端程序需要维护host和drive之间的元素据。
3. 缺乏系统支持。现有工作没有考虑多个进程共享drive的情况，对数据保护、进程隔离性和带宽调度都没有考虑。
4. 在drive中使用嵌入式ASIC的方案缺点在于ASIC太过于专有化，1块ASIC只能支持1种特定的任务，缺少灵活性。



### INSIDER设计

##### 1. 基于FPGA的ISC单元

ISC单元应该满足如下要求：高可配置性，与上述ASIC相反；2.  支持大量的并行；3. 高能效。在此基础上作者对比了几类硬件，如Table 1所示，得到结论FPGA最合适。![image-20221207195626034](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212071956073.png)

##### 2. Drive 架构 

1. 控制平面和数据平面分离。Controller主要包括两部分，fireware与ISC runtime、drive相配合完成控制层的请求，将host请求中的逻辑地址转为物理地址， 并请求存储单元了。Accelerator集群分离到数据平面，拦截存储单元和DMA控制器之间的数据，并进行计算。

   控制、数据平面分离：accelerator集群不参与控制逻辑，只进行数据计算。控制层需要负责host端的文件按访问权限检查、发起请求等。accelerattor上的计算逻辑也不会破坏控制层的相关部件。

2. Accelerator集群。如Figure4所示每个Accelerator分为2层，内层为可编程区域，每一个application slot可以存放1个用户定义的加速程序，多个app slot共享FPGA的不同硬件部分，因此是在空间上并行。外层是硬件运行环境负责，用户不可编程，负责流控制和将数据分配到对应的slot上。

![image-20221207200158432](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212072001504.png)

##### 3. Host端编程模型

Host端的编程模型为Virtual file abstraction。对虚拟文件的读写操作可以透明的转化为对真实文件的读写并触发相关的ISC计算。其主要组成有Table2中的文件读写方法。Host端的代码示例如Listing1所示

![image-20230109191644434](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091916482.png)![image-20230109201146752](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301092011802.png)

1. 虚拟文件读

   虚拟文件读缓解drive —> host方向的带宽瓶颈问题。Listing 1为host端的示例代码。

   1. 系统启动阶段为每一个用户创建一个映射文件：.USERNAME.insider，用于虚拟文件和真实文件之间的映射。
   2. Registration通过调用reg_virt_file确定用户要读取的数据，需要1.真实文件地址，2accelerator  ID两个参数，并将两个参数映射成一个虚拟文件，映射关系记录在上述的映射文件中，对应的accelerator程序放入drive的slot中。
   3. 通过vopen API打开虚拟文件。
      1. runtime先读取映射文件获取真实文件的地址
      2. runtime访问host文件系统判断访问权限
      3. 检查通过后返回正确的文件描述符，并将Registration阶段登记的slot号传入drive
      4. runtime请求INSIDER内核部分为真实文件打赏append—only标记，防止读取过程中被修改。
      5. INSIDER使用filefrag命令获得文件在存储设备山的分布情况(逻辑地址)并传入drive
      6. 将必要的运行时参数传入drive
   4. vread API对虚拟文件进行读。INSIDER读取真实文件的内容，accelerator截取数据并进行相应处理，输出通过DMA传入host。在host端看就好像读取一个普通的文件。
   5. vclose API关闭虚拟文件，清除append-only标记和缓冲区等运行时资源。

2. 虚拟文件写。

   虚拟文件读缓解host—> host方向的带宽瓶颈问题。其工作与读基本类似。

3. 并发控制

   并发情况：1. vwrite和vread 2. vwrite和vwrite 3. vread和normal write（前述vread打了append-only的标记，所以还是可以写的，为什么没有read和vwrite的冲突？）

   采用Linux对文件的锁API对文件实现并发控制，由用户自行使用。

   ![image-20230109195626767](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091956836.png)

​                      

##### 4. Drive端编程模型

INSIDER为processor代码提供了接口，因此只需关注计算逻辑，drive-side编译器，支持c++和RTL。为Accelerator提供的接口实际就是三个FIFO，数据以flit的形式读取，即APP_DATA结构体。3个FIFO：input FIFO存储拦截的待加速的数据、output FIFO存储处理完要被写回的数据，parameter FIFO存储host发送的运行时参数。输入输出数据被包装成APP_Data的形式，len用来指示最后一个flit的大小                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               

![image-20221212110348030](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212121103279.png)

##### 5. 系统流水线

naive版本的时间由$t = t_{drive_read} +t_{drive_comp.} + t_{out put_trans.} + t_{host_comp}$计算, 流水线下的时间
$max(t_{drive_read}, t\_{drive_comp}., t_{out put_trans.}, t_{host_comp})$。作者设计了INSIDER硬件逻辑，确保$t_{drive_read}和t_{drive_comp.}$其完全流水线化。重叠drive、bus、host时间靠2步：1.在vopen的时候就开始提前计算，因为在vopen时drive就有真实文件的全部信息。2.分配host内存存储提前计算好的结果。这样每次主机端进行vread只需从存储区弹出预先计算好的数据。

##### 6. 自适应带宽调度器

drive同时支持多个application面临两个问题：

1. 通过多路技术将数据转发到正确的accelerator
2. 根据不同的accelerator的数据处理率动态调整为不同accelerator分配的带宽。

解决上一段多路复用的问题。增加了ISC和MUX两个器件和SIF队列。MUX接受来自original  fireware和ISC的请求，1.转发请求到存储单元读取数据，2.将对应的accelerator ID  push到SIF(FIFO)中，其中slot 0永远用于普通读请求。

动态带宽调整根据信用机制，在ISC unit中credit寄存器集合，根据每个slot对应的data FIFO中剩余空间来动态维护credit 寄存器，从而动态调整带宽。

![image-20221212164419840](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202212121644880.png)



### INSIDER 实现

1. drive原型

   INSIDER drive的实现。如Figure8所示。 基于PCIe和FPGA，主要三部分，storage controller、drive controller和DMA controller， 由于市面上FPGA没有足够的闪存接口，因此使用DRAM作为存储设备。另外存储控制器和DMA控制器中的delay and throttle unit(具体作用？控制速度？)可以在host端配置参数。

![image-20230109184118610](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091841650.png)

2. INSIDER软件栈
   1. Compiler。INSIDER为host和drive都提供了编译器，其中Host端是g++编译器，drive编译器的前端为LLVM，后端由Xilinx Vivado HLS和自建RTL代码转换器实现。
   2. Software Simulator。INSIDER提供一个软件仿真器，可以避免FPGA程序数小时的便宜运行时间。
   3. Host-side runtime library。在用户空间实现，绕过OS的IO栈，包含Table 2中的所有方法和配置drive参数的方法，并提供流水线支持。
   4. Linux kernel drivers。提供了两套驱动器，其中第1个驱动器将drive作为普通的存储设备，第二个驱动器与ISC相关。



### 评估

1. 实验环境

   评估采用32核CPU

   ![image-20230109183903314](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091839351.png)

2. 在具体的Application场景下测试INSIDER

   如Table3所示为先前ISC工作中用于测试的Application，作者在INSIDER中实现C++版本来评测INSIDEER性能。

   ![image-20230108181750480](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301081817559.png)

   Figure 9是针对baseline的速度提升柱状图，其中

   	1. baseline是最原始的host-only版本。
    	2. Host-bypass是在host-only版本上，同时采用ISC接口和PT程序(不进行任何计算的accelerator程序)绕过host IO栈。
    	3. Host-bypass-pipeline在上述基础上对计算时间和读入时间进行流水线。
    	4. INSIDER
    	5. x8/x16表示PCIe Gen3 x8 and x16

   ![image-20230109183238962](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091832004.png)

   INSIDER性能提升主要有3方面：

   	1. IO stack速度提升；
    	2. ISC任务下发和流水线；
    	3. 数据量减少。

   Figure10, 对各方面带来的速度提升进行进一步分解。当PCIe x8时bus带宽不够，数据量减少是速度提升的主要原因。x16时bus带宽充足，数据量减少带来的速度提升效果有限，另外两方面是速度提升的主要因素。

   ![image-20230109183728930](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091837989.png)

3. 使用INSIDER的开发成本

   Table3中的LoC为代码行数，Devel.Time为开发时间，可以看出相比于其他ISC系统，INSIDER极大降低开发成本，其剩余的主要开发时间只在开发drive中的accelerator程序。

4. 并发处理

   解释Figure11，测试多个application/accelerator同时执行的drive带宽调度效果。测试程序的输出速率和drive带宽如图注所示，drive带宽小于测试程序输出速率的和。测试方法为修改host端代码使得host端不进行计算，因此host测得的总执行时间约等于drive端的执行时间，再用公式$rate = ∆size(data)/∆time$计算。从Figure 11可以看出INSIDER动态公平的调度各个application的带宽。

   ![image-20230109182733197](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091827247.png)

5. 资源使用情况分析

   FPGA资源只要有三方面：

   	1.  accelerator程序逻辑。
    	2.  INSIDER框架部分消耗的资源
    	3.  用于IO的IP核，例如DRAM控制器和DMA控制器。

   从Figure 6可以看出，第3方面的资源消耗最多，但该部分在最新的存储设备已经有嵌入包含(不属于INSIDER部分)，因此最终的资源消耗只考虑1、2两方面。
   对比高端(XCVU9P)和低端(XC7A200T)的FPGA芯片，在最好情况下，低端的FPGA芯片都可以同时支持文中的5个accelerator程序。

   ![image-20230109182157590](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091821647.png)

6. 与基于ARM系统的比较

   ​	比较基于ARM和FPGA的ISC系统性能的方法如下：将基于FPGA的ISC单元替换为ARM，其他不变。使用上文用到的Application。依旧采用流水线设计，因此总时间为$T_{e2e} = max(T_{host} , T_{trans}, T_{ARM})$，其中$T_{host}$和$ T_{trans}$因为与ARM无关，采用之前INSIDER测得的数据。使用Cortex-A72，四核三向的高端超标量ARM处理器

   ​	Figure12，INSIDER平均性能提升达到12倍。性能提升和计算机密集度有关，其中KNN计算密集度最高，性能提升也最明显。

   ![image-20230109181745303](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091817351.png)

   ​	对比ARM和INSIDER的经济效益，按每美元的数据处理量对比。平均情况下，INSIDER经济效益比ARM高31倍。

   ![image-20230109181818189](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202301091818237.png)































