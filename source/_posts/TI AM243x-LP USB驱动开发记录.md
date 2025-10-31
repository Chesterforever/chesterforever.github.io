---
title: TI AM243x-LP USB驱动开发记录
date: 2025-10-31 23:41:05
tags: [Soc]
index_img: /img/avatar.png

---

# TI AM243x-LP USB驱动开发记录

不得不承认之前被ST的软件生态养的太好了，第一次用TI的Soc开发板，踩了很多坑，但是狠狠恶补了一波多核和Soc的知识点，不亏。

## 相关资料

TI官网上给开发者提供的相关资料有

- AM243x芯片手册
- AM243x-LP开发板硬件手册
- SDK（最新版本SDK分为mcu+sdk，motor control sdk，industrial sdk三种）
- SDK使用手册

SDK使用手册直接用浏览器打开然后用浏览器的翻译功能就行,芯片手册和开发板硬件手册只有英文版pdf，所以我用了Zotero+pdf2zh插件进行翻译（这个是github开源的pdf翻译方案，而且有双语对照功能，还是挺好用的）

我把翻译好的芯片手册和开发板硬件手册的链接放在下面



## 踩坑过程

#### BOOT过程

因为网上关于TI开发板的视频资料较少，所以基本上是直接根据SDK使用手册一步一步做的，这就导致了一旦使用手册里有写的不明确的地方，就要在这些地方卡一段时间去试错。

我遇到的第一个问题是芯片如何正确启动并初始化。就像其他的高级Soc一样，TI为AM2434的启动设置了多阶段引导加载程序，也就是多级bootloader。

{% asset_img 0f634769-308d-424b-a8dc-9764f103067d.png  %}

首先是第一级bootloader，出于安全考虑，TI在芯片出厂前将其保存在安全的只读内存ROM中，称为ROM引导加载程序（RBL）。RBL程序负责引导进入下一级bootloader，而不需要知道芯片的其他内核。

然后是第二级bootloader，主要负责处理不同内核上的复杂引导加载，称为SBL（second bootloader）。加载多核应用程序比加载单核应用程序要复杂很多，要考虑映像准备和共享内存访问等问题。好在TI提供了相关的makefile文件`makefile_ccs_bootimage_gen`，方便用户自定义编译出需要的SBL文件。

在了解完这些前置知识点后，我就老老实实一步一步跟着SDK使用手册的步骤。

对于AM243X-LP开发板来说，TI提供了多种启动方法（通过UART,SD卡或CCS脚本）需要注意的是这里的启动只需要我们将TI提供的SBL想办法放到芯片的二级启动地址下，对应使用手册中的命令

```
 cd ${SDK_INSTALL_PATH}/tools/boot
  python uart_uniflash.py -p COM<x> --cfg=sbl_prebuilt/am243x-lp/default_sbl_null.cfg
```

然而我在执行这一步时就卡住了，问题现象时是正常执行第一条命令，在执行第二条命令时会卡住并应执行超时而失败，失败原因是XMODEM协议校验不通过

{% asset_img f55d9c0c2143326082f46bf8f2b85083.jpg  %}

于是我尝试了其他的启动方案比如用CCS脚本，甚至为此去下载了TI官方的烧录工具uniflash，最后也没能成功。没办法，只能去TI的论坛下面求助，那天刚好是星期五，周末TI不上班，最后星期二才回我

{% asset_img da0a6b92-1163-467f-af45-1a8772a4f39f.png  %}

这个时候我才知道原来TI的这款芯片还分了三种安全等级，分别对应

- GG 非安全   适用固件后缀为无后缀
- HS 高安全   适用固件后缀为_hs
- FS 现场安全   适用固件后缀为_fs

除了看芯片丝印信息这一方法来得知安全等级信息，在逛TI论坛时我还发现了一个方法来看自己的芯片是GP还是HS_FS，这里就不多介绍了，放个链接[[FAQ\] [参考译文] [常见问题解答] TDA4VM：如何检查器件类型(GP FS)？ - 处理器（参考译文帖）(Read Only) - 处理器（参考译文帖） - E2E™ 设计支持](https://e2echina.ti.com/support/machine-translation/mt-processors/f/mt-processors-forum/1010320/faq-tda4vm-gp-fs?tisearch=e2e-sitesearch&keymatch=GP#)

但其实从22年9月后TI就逐渐取消了对GP型号的生产和支持，SDK在9.0版本后就完全不支持GP型号了，所以基本上现在市面上买到的器件都是HS_FS型号的，所以不用担心这个问题，直接用最新版的SDK就行。

而我的芯片由于是GP型号的，只能去官网上下载了08030018版本的SDK和1.12.1版本的sysconfig工具，并导入到CCS的product中。如果是HS_FS型号的开发板，版本下载官网最新的就行。

在经历了这些后，我终于在老版本的SDK上找到了后缀不带.hs_fs的SBL文件并成功复现了SDK使用手册中的步骤。这个时候就可以将LP启动模式切换到OSPI模式了，然后跟着使用手册的步骤应该没有什么坑了，遇到问题就问AI，大概率能解决。执行完OSPI boot之后，就可以直接用CCS直接烧录和调试程序了，不需要每次切换回UART boot模式。

> > 在终端执行命令行时尽量用相对路径而不用绝对路径，这样能有效避免应命令引用路径过长导致的问题
>
> > 如果是HS_FS型号的开发板，要装OpenSSL，当编译好一个程序镜像后，TI 的构建流程会调用 OpenSSL 来生成一个数字签名（就像给软件盖一个加密的印章）。芯片上的 ROM 引导程序或二级引导程序在加载应用前，会使用预先烧录在芯片中的公钥来验证这个签名。如果验证失败，说明软件可能被篡改，芯片将拒绝执行，从而防止恶意代码运行

#### 示例代码的调试与运行

示例代码的使用在使用手册中的开发者指南有详细的教学，包括怎么导入，编译和烧录。

主要讲一下烧录，只需将tool文件夹复制到工作区，然后修改default_sbl_ospi.cfg文件，将第四个命令的file对象改为自己编译出来的.appimage文件，第五行命令是执行XIP image文件的，直接注释就行

{% asset_img 4ac3ee73-0cc1-41e3-80f7-7b88a189df08.png  %}

> **XIP Image** 的全称是 **eXecute-In-Place Image**，中文可译为 **“就地执行映像”** 或 **“直接执行映像”**。
>
> 它是一种**存储在非易失性存储器（如 Flash）中，且能够被 CPU 直接从中读取并执行的程序二进制文件**，而无需将其先复制到 RAM（内存）中，这个是由链接脚本决定的。
>
> 用于快速启动，节省RAM空间，但是执行速度较慢，写入困难

在debug的时候能注意到左边的线程中有很多内核对象，我查了一下它们的命名规则

{% asset_img f6cd1f99-f18d-4784-a251-4ff17ddb3ddd.png  %}

```
MAIN_Cortex_R5_0_0  // 集群0，核心0
MAIN_Cortex_R5_0_1  // 集群0，核心1  
MAIN_Cortex_R5_1_0  // 集群1，核心0
MAIN_Cortex_R5_1_1  // 集群1，核心1
DMSC_Cortex_M3_0    // 设备管理安全控制器
ICSS_G0_PRU_0       // 可编程实时单元0
ICSS_G0_RTU_PRU_0   // 实时单元PRU0
ICSS_G0_TX_PRU_0    // 发送PRU0
ICSS_G0_PRU_1       // 可编程实时单元1
BLAZAR_Cortex_M4F_0 // 通用处理核心
```

最后放一下执行USB CDC示例代码后的运行效果

{% asset_img d4798e5b0d2a2a31b548aa669d8c6a5a.png  %}





#### 将示例代码从R5F内核移植到M4F内核上

（持续更新中）
