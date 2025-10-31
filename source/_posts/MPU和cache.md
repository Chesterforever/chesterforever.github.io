---
title: MPU和cache
date: 2024-09-20 19:03:24
tags: [MCU,STM32H7]
index_img: /img/avatar.png
---

# **MPU和cache**

为什么要有MPU：阻止用户应用破坏关键任务；阻止代码注入攻击；改变内存访问属性；



cache的作用：以H7为例，内核480M，RAM只有240M，通过cache就能实现类似倍频的效果；



MPU一般有16个域，每个域配置8个等大小的子域（子域最小大小受cache line影响，禁止子域就是禁止该子域的MPU配置）优先级值越大，优先级越高，当域存在重叠时，以优先级高的域的配置为主；



内存类型分为下面三种类型

Normal memory：由DRAM，SRAM，FLASH等通用存储介质组成，支持cache缓存，预取，允许乱序访问，缓冲合并，充分发挥了各种机制，所以性能最强；

Device memory：内存映射外部寄存器，不可缓存，不可预取，严格顺序访问，禁止写合并，性能稍差；

Strongly-ordered Memory：系统关键组件，不可缓存，不可预取，绝对顺序执行(严格按照指令顺序执行)，禁止任何缓冲，所有访问全局可观察，性能最差，访问延迟最长；



**MPU配置：**  

内存属性

XN：执行禁止，标记内存区域是否允许执行指令，默认使能即可；

AP：访问权限，决定CPU特权级和用户级的读写权限；

TEX:拓展内存类型；

S：是否支持多核/总线主设备共享；

C：数据是否可缓存；

B：写操作是否允许缓冲；

{% asset_img 0592f54a-de70-4e12-95e2-cc8cfc6e6b74.png  %}

![](2024/09/20/MPU和cache/0592f54a-de70-4e12-95e2-cc8cfc6e6b74.png)

配置为1011（normal   not shareable）时性能最强



##### Cache

- 为什么要有cache？

  根据局部性原理，如果某个数据被访问了，那个它自身以及临近的数据都很有可能在不久的将来被访问，所以计算机决定把这些数据从慢速的主存放到快速且小的存储单元中，这个单元就是cache

- cache是如何工作的

  当CPU需要读取数据时，先会查询要访问的内存地址，cache控制器检查这个地址的数据是否已经在cache中，如果检查到存在则直接从cache中将数据返回给CPU，如果检查到不存在触发cache miss，具体步骤为CPU先从主内存DRAM读取包含目标地址的整个数据块（为什么这样读取涉及映射策略），将数据块载入cache，并且用替换算法淘汰cache的旧块，淘汰旧块时检测旧块是否被CPU修改过（dirty bit=1），若被修改过则需执行write back操作

  通过这一系列的操作将数据从cache返回至CPU中



内部FLASH仅需开启指令cache即可达到最高性能；

外部QSPI FLASH开启读cache和写cache即可达到最高性能；

DTCM和ITCM主频和CPU一样，无需配置MPU和cache；

涉及到ADC+DMA读取或串口+DMA收发，在开启DMA前先clean，在读取DMA后invalidate；

涉及到DMA的数据一致性问题解决思路：在输出层调用SCB_CleanInvalidateDCache（）；(一般不建议用这个函数，因为会带来性能损失的问题)

SCB_InvalidateDCache_by_Addr：将指定地址与指定大小的Ram的内容同步到Cache中，典型使用场景为DMA接收。
SCB_CleanDCache_by_Addr：将指定地址与指定大小的Cache的内容同步到Ram中，典型使用场景为DMA发送。



**使用函数SCB_InvalidateDCache_by_Addr，SCB_CleanDCache_by_Addr等函数注意事项**

**addr ： 操作的地址一定要是32字节对齐的。**

**dsize ：一定要是32字节的整数倍**



