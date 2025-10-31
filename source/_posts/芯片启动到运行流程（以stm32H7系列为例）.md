---
title: 芯片启动到运行流程（以stm32H7为例）
date: 2024-04-25 23:03:00
tags: [MCU]
index_img: /img/avatar.png

---

# **芯片启动到运行流程（以stm32H7系列为例）**

1.芯片供电域（看芯片手册）

2.上电启动流程（从硬件上看）

上电复位和手动复位都是操作电容

{% asset_img image-20250811130212246.png  %}



{% asset_img image-20250811130619576.png  %}



从图上看感觉就是各种时钟的初始化

3.软件启动流程

启动文件路径（Drivers->CMSIS->Device->ST->STM32H7xx->Source->Templates->各种种类启动文件如iar mdk gcc）

中断向量表（重点）

4.整个工程启动流程

5.加载域 运行域     .map文件   .html文件