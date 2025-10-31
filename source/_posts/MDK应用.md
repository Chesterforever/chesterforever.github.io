---
title: MDK应用
date: 2025-06-20 15:13:24
tags: [MCU,debug]
index_img: /img/avatar.png
---

**MDK应用**

JTAG(20PIN) SWD(5PIN)

可以在debug时看全局变量 局部变量 在memory里看数据

> Hardfault

硬件异常 内存异常 总线异常 处理异常

触发不同异常进入的各种中断可以配置优先级（实战好像没什么用）

本质就是通过硬件寄存器检测数据对不对，不对就触发中断.

{% asset_img image-20250811153714902.png  %}

进入异常后，判断LR寄存器bit2，如果等于0就把MSP给R0，如果等于1就把PSP给R0，最后进入异常处理函数

总结：看PC指针

