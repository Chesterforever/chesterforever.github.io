---
title: MCU启动流程
date: 2024-09-22 15:13:24
tags: [MCU]
index_img: /img/avatar.png
---

#### **MCU启动流程**

{% asset_img e2939426-7334-437f-be36-495cb5ce0872.png  %}



{% asset_img f74c7b9f-0983-4b8b-8e20-6b6cb6c51d88.png  %}



{% asset_img a7146b0a-71c9-43e8-bb9c-c741b93c7b74.png  %}



以cm3和cm4为例（cm7会有不同）

上电复位后硬件强制PC指向自举区的入口地址，先执行自举区代码（初始化基础时钟->通过boot引脚检测启动模式），如果（boot0=0，boot1=0）从主flash启动，那么就会将用户flash（0x08000000）重映射到0x00000000中，如果（boot0=1，boot1=0）从系统存储器启动，那么就会跳转到系统bootloader，如果（boot0=1，boot1=1）从SRAM启动           还有一种特殊情况就是使用了选项字节，这个时候就不看boot引脚状态

如果从主flash启动，就从0x00000000读MSP，0x00000000读PC（PC此时指向reset_handler地址，由于重映射关系，该地址一般是0x08000004之后的某一地址），读完地址就会跳转到该地址执行reset_handler（初始化.data,.bss,系统时钟......）最后执行main（）

如果从系统存储区启动，就会将系统存储器物理地址重映射到0x00000000上，然后跳转到系统bootloader（根据自举区固化好的代码，这个代码存储了系统bootloader的入口地址），bootloader会先初始化（初始化外设通信，配置外设时钟，初始化GPIO，初始化flash编程接口），然后bootloader会进入一个主循环（在这个循环中发送或等待特定协议用于监听主机发送命令），接收到主机各种命令后会进行处理（如擦除flash，接收bin/hex文件数据写入用户flash地址里，读flash，......），最后会执行用户程序reset_handler（初始化.data,.bss,系统时钟......）最后执行main（）

如果从SRAM启动，其实就是debug过程会执行（）

在复位处理函数中，先初始化堆栈指针（将从向量表加载的初始SP值设置到硬件堆栈指针寄存器中），然后初始化.data段（存放已初始化的全局变量和静态变量）清零.bss段，再初始化系统时钟（调用systeminit函数）初始化浮点运算单元和MPU，最后跳转到main（）函数