---
title: bootloader
date: 2025-03-05 00:18:45
tags: [MCU]
index_img: /img/avatar.png
---

# **bootloader**

> 单片机启动流程

首地址是MSP

中断向量表

> 跳转配置

- 将APP程序加载到支持运行程序的Flash或者RAM中

- 复位RCC时钟
- 复位所有开启的外设
- 关闭滴答
- 关闭所有中断
- 设置跳转PC，SP和Control寄存器
- 如果用了RTOS，则要设置为特权级模式，使用MSP指针

> APP程序问题

- APP程序入口依然是复位中断服务程序
- 注意设置APP的中断向量表地址
- BOOT占用的RAM空间可以全部被APP使用
- APP程序版本号和程序完整性问题（CRC或MD5校验）
- 固件加密
- APP调回BOOT，使用NVIC_SystemReset软件复位

> 调试下载

- 将BOOT下载进去猴，就可以传统方式调试APP的程序
- APP或BOOT程序排查问题

> 移植步骤

- 编译器生成.hex文件

- 移植output.hex和hex2bin文件

  `这块有些问题`

要确定下载媒介是什么（eMMC，SD卡，U盘）

> 加密算法

AES加密（密钥）