---
title: FreeRTOS内核任务链表与状态切换(一)
date: 2025-03-05 02:03:24
tags: [RTOS]
index_img: /img/avatar.png
---

# **FreeRTOS内核任务链表与状态切换(一)**

首先创建(静态)任务会在SRAM里的.bss段的FreeRTOS_HEAP_SIZE里生成一个TCB块

{% asset_img 1a23e6ef-621e-49d3-b9b3-37c3570516bb.png  %}

TCB块里会有一个结构体，这个结构体存储了以下数据

{% asset_img 808b660a-7f3d-4e6a-bf66-39a950f7707a.png  %}

pxTopOfStack指向该任务的栈顶（通过PSP指针初始化栈寄存器）

*pxStack指向该任务的栈底

比较关键的是xStateListitem和xEventListitem这两个成员

xStateListitem主要挂载到os调度有关的链表（就绪，延迟，等待删除，挂起）

xEventListitem主要挂载到Queue相关的链表

{% asset_img d936a597-ac51-41fe-8867-a88924375b84.png %}

os链表如图所示，其中就绪链表的数量与任务优先级数量有关，每个优先级对应一个就绪链表（由uxPriority决定，优先级数字越大，越早进行调度）

所以内核调度（PendSV）时会有一个pxCurentTCB指针，这个指针先顺着链表找，当它找到想要的成员（xStateListitem或xEventListitem）时，它就会顺着指向这个任务的TCB块

但实际上如果按上图所说链表长这样，那肯定会有很多空间被浪费，所以实际上的架构如下图所示

{% asset_img 00e8822f-7373-4d38-92ea-100e7e02ef36.png  %}

当FreeRTOS被初始化完后，与os调度相关的链表（与xStateListitem有关）就作为全局静态变量存在os的堆栈空间里

而Queue链表则作为局部变量是在Queue初始化时创建

每个节点就是上图中的结构体Listitem_t，只和自己节点的上一个节点和下一个节点有关

xListEnd其实是链表里的一个哨兵节点