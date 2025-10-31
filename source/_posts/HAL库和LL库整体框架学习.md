---
title: HAL库和LL库整体框架学习
date: 2023-11-05 09:05:24
tags: [MCU]
index_img: /img/avatar.png
---

**HAL库和LL库**（可以混合使用）

hal库：函数复杂，但是直接对寄存器调用，移植方便

LL库：使用很多内联函数（省去进入和退出函数的时间）高效速度快

**hal库：**

以_IT为后缀结尾表示中断方式

用cubemx配高速时钟注意检查晶振大小要相同

 HAL_Init:配置NVIC 更新全局变量SystemCoreClock（此时用的还是内部HSI48M时钟）

SystemClock_Config:配置时钟电压范围（不同电压对应不同主频）配置时钟结构体

​                                     更新时钟HAL_InitTick(此时用的是更新后时钟)

 

学习这些库的目的也是为了方便调用，比如用hal库初始化，然后自己写寄存器操作代码造点轮子，最后就能很快地搭建起整个代码框架