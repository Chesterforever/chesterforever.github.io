---
title: 被CANFD问题硬控一整天后有感
date: 2025-10-18 11:45:11
tags: [MCU,CAN]
index_img: /img/avatar.png
---



# 被CANFD问题硬控一整天后有感

## 前言

最近拿到一个CANFD示教按键的需求，需要实现检测按键长按3s后通过CAN总线发一个报文的功能。本来以为这是一个简单的任务，没想到最后被CANFD数据段高波特率下的数据传输问题硬控了一整天。还好最后解决了,不然晚上真的睡不好觉。

## 方案说明

总体思路还是对按键检测和CAN收发两个功能实现模块化封装，方便之后可能的移植需求

### 1.按键检测模块

这个功能有很多种实现的方法，主要分为GPIO外部中断和定时器中断轮询这两种方法。这里让AI对比了一下两种方案的优劣

| 特性对比       | **定时器中断轮询**                                           | **GPIO外部中断**                                             |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **核心机制**   | 定时器周期性触发中断，在中断服务程序（ISR）中**主动扫描**所有按键的状态 | 按键电平变化（如按下/释放）时**主动触发**中断，CPU再响应     |
| **CPU占用**    | **相对较高**。即使无按键操作，定时器中断仍会周期性发生，占用CPU时间片。 | **极低**。无按键动作时，CPU完全不被干扰，可处理其他任务或进入低功耗模式。 |
| **响应速度**   | **固定延迟**。响应时间取决于扫描周期（如10ms），存在最大为一次扫描周期的延迟。 | **近乎实时**。电平变化瞬间即可触发响应，延迟极短。           |
| **抗干扰能力** | **较强**。可在软件扫描逻辑中实现稳定的消抖算法，可靠性高。   | **对硬件设计敏感**。易受硬件抖动和噪声影响，需在硬件（RC滤波）或软件（中断内延时判断）层面做消抖。 |
| **多按键扩展** | **易于扩展**。单个定时器即可扫描多个按键，对硬件引脚无特殊要求。 | **受限于硬件资源**。每个按键需占用一个支持外部中断的GPIO引脚，且某些MCU中多个引脚可能共享一个中断向量，需在ISR内二次判断。 |
| **适用场景**   | 多按键系统（如键盘矩阵）、对实时性要求不极端、需要稳定可靠消抖的应用。 | 按键数量少、对响应速度和低功耗有极高要求的应用（如电池设备、游戏手柄）。 |

考虑到硬件电路中按键对应的输入GPIO不支持外部中断，所以我采用的方案是定时器中断轮询，这个方案的劣势在于定时器会一直触发按键状态机轮询检测，CPU占用过高且无法实现低功耗，不过还好已知后续这个项目的拓展不会特别复杂，所以经过评估后使用这个方案。

除了考虑检测到按键按下的方案，我们还要考虑按下之后的3s长按检测，这个3s的时间检测可以用另一个定时器计时，也可以用系统时钟计时。我采用的是另一个定时器计时（这里由于项目不复杂，其实还是用系统时钟计时更好）

有了大致方案思路后就可以开始代码编写了，新建key.c和key.h放到中间层文件夹下，里面主要实现按键状态机的轮询和相关按键状态下的消抖算法。然后第一个定时器中断中只要调用轮询的函数，第二个定时器中断中只要调用CANFD的发送函数，这样就很好地实现了模块的封装和接口的调用。

### 2.CANFD收发模块

要实现的功能主要是CANFD的初始化（包括硬件过滤器和全局过滤器的初始化），CANFD的发送函数，CANFD的接收回调函数这三个。为了方便之后的移植，我将每路CAN封装成实例，在初始化时分别对每路CAN进行初始化，并用指针数组加上CANID偏移指向这路CAN的句柄,这样之后使用CAN时只需要用instance_id来表示使用的是哪路CAN即可

```
/**
  * @brief  初始化MyCAN模块
  * @author  xujiawei
  * @param  hfdcan: 指向FDCAN_HandleTypeDef结构的指针（指向HAL库的CAN句柄）
  * @param  instance_id: 实例标识符，用于指定要操作的多路CAN中的某一实例（0=CAN1, 1=CAN2）
  * @retval MyCAN_Status_e 初始化状态
  */
MyCAN_Status_e MyCAN_Init(FDCAN_HandleTypeDef *hfdcan, uint8_t instance_id)
{
    if (hfdcan == NULL || instance_id >= MYCAN_MAX_INSTANCES) {
        return MYCAN_ERROR;
    }

    MyCAN_Handle_t *p_can = &can_handles[instance_id];
    p_can->phfdcan = hfdcan;

    /* 初始化该实例的接收FIFO */
    memset((void*)p_can->rx_fifo, 0, sizeof(p_can->rx_fifo));
    p_can->rx_write_index = 0;
    p_can->rx_read_index = 0;
    p_can->rx_count = 0;
    p_can->rx_callback = NULL;
    p_can->tx_callback = NULL;

    HAL_FDCAN_ConfigTxDelayCompensation(hfdcan, hfdcan->Init.DataPrescaler * hfdcan->Init.DataTimeSeg1, 0);
    HAL_FDCAN_EnableTxDelayCompensation(hfdcan);

    /* 使能接收FIFO中断 */
    if (HAL_FDCAN_ActivateNotification(p_can->phfdcan, FDCAN_IT_RX_FIFO0_NEW_MESSAGE, 0) != HAL_OK) {
        return MYCAN_ERROR;
    }

    /* 1. 配置一个硬件过滤器：设置为接收所有ID */
    FDCAN_FilterTypeDef sFilterConfig;
    sFilterConfig.IdType = FDCAN_STANDARD_ID;        // 此设置对标准和扩展ID都有效，当掩码为0时
    sFilterConfig.FilterIndex = 0;                   // 选择一个过滤器组，例如0
    sFilterConfig.FilterType = FDCAN_FILTER_MASK;    // 使用掩码模式
    sFilterConfig.FilterConfig = FDCAN_FILTER_TO_RXFIFO0; // 指定匹配的报文存入RX FIFO0
    sFilterConfig.FilterID1 = 0x0000;               // 期望接收的ID，可设为任意值（因为掩码为0）
    sFilterConfig.FilterID2 = 0x0000;               // **关键：将掩码设置为0，表示不检查任何位**

    if (HAL_FDCAN_ConfigFilter(p_can->phfdcan, &sFilterConfig) != HAL_OK) {
        return MYCAN_ERROR;
    }

    /* 2. 配置全局过滤器：接受所有不匹配的帧 */
    if (HAL_FDCAN_ConfigGlobalFilter(p_can->phfdcan,
                                     FDCAN_ACCEPT_IN_RX_FIFO0, // 接受所有不匹配的标准帧到FIFO0
                                     FDCAN_ACCEPT_IN_RX_FIFO0, // 接受所有不匹配的扩展帧到FIFO0
                                     FDCAN_REJECT_REMOTE,      // 通常拒绝远程帧
                                     FDCAN_REJECT_REMOTE) != HAL_OK) {
        return MYCAN_ERROR;
    }
    
    /* 启动CAN外设 */
    if (HAL_FDCAN_Start(p_can->phfdcan) != HAL_OK) {
        return MYCAN_ERROR;
    }

    p_can->is_initialized = 1;
    return MYCAN_OK;
}
```

这里还是得介绍一下为什么要用注册的方式实现回调函数而不是直接把APP层的回调函数放到HAL_FDCAN_RxFifo0Callback(FDCAN_HandleTypeDef *hfdcan, uint32_t RxFifo0ITs)中，其实主要就是一种依赖倒置的体现，HAL库定义抽象接口，APP层提供具体实现，从而实现了“控制反转”，很好地体现嵌入式系统设计中模块化和解耦的核心思想。

```
/**
  * @brief  注册接收回调函数
  * @param  instance_id: CAN实例ID
  * @param  callback: 回调函数指针
  */
void MyCAN_RegisterRxCallback(uint8_t instance_id, MyCAN_RxCallback_t callback)
{
    if (instance_id < MYCAN_MAX_INSTANCES) {
        can_handles[instance_id].rx_callback = callback;
    }
}

```

```
/**
  * @brief  接收FIFO中断回调函数
  * @param  hfdcan: CAN句柄指针
  * @param  RxFifo0ITs: 中断标志
  */
void HAL_FDCAN_RxFifo0Callback(FDCAN_HandleTypeDef *hfdcan, uint32_t RxFifo0ITs)
{
    if ((RxFifo0ITs & FDCAN_IT_RX_FIFO0_NEW_MESSAGE) != RESET) {
        /* 遍历所有实例，找到是哪个CAN实例触发的中断 */
        MyCAN_Handle_t *p_can = NULL;
        for (int i = 0; i < MYCAN_MAX_INSTANCES; i++) {
            if (can_handles[i].phfdcan == hfdcan && can_handles[i].is_initialized) {
                p_can = &can_handles[i];
                break;
            }
        }

        if (p_can == NULL || p_can->rx_count >= MYCAN_RX_FIFO_SIZE) {
            return; /* 未找到对应实例或FIFO已满 */
        }

        FDCAN_RxHeaderTypeDef rx_header;
        MyCAN_Message_t new_msg = {0};

        if (HAL_FDCAN_GetRxMessage(hfdcan, FDCAN_RX_FIFO0, &rx_header, new_msg.data) == HAL_OK) {
            /* 解析消息头 */
            new_msg.id = rx_header.Identifier;
            new_msg.id_type = (rx_header.IdType == FDCAN_STANDARD_ID) ? MYCAN_STANDARD_ID : MYCAN_EXTENDED_ID;
            new_msg.dlc = rx_header.DataLength;
#if (MYCAN_USE_FD == 1)
            new_msg.brs = (rx_header.BitRateSwitch == FDCAN_BRS_ON) ? 1 : 0;
            new_msg.fdf = (rx_header.FDFormat == FDCAN_FD_CAN) ? 1 : 0;
#endif
            new_msg.timestamp = rx_header.RxTimestamp;

            /* 将消息存入该实例的FIFO */
            memcpy(&p_can->rx_fifo[p_can->rx_write_index], &new_msg, sizeof(MyCAN_Message_t));
            p_can->rx_write_index = (p_can->rx_write_index + 1) % MYCAN_RX_FIFO_SIZE;
            p_can->rx_count++;

            /* 调用该实例的用户回调函数 */
            if (p_can->rx_callback != NULL) {
                p_can->rx_callback(&new_msg);
            }
        }

        /* 处理完成后，重新激活该实例的RX FIFO0新消息中断 */
        HAL_FDCAN_ActivateNotification(hfdcan, FDCAN_IT_RX_FIFO0_NEW_MESSAGE, 0);
    }
}

```

```
/**
  * @brief  APP层接收回调函数
  */
void MyApp_CAN2_MessageHandler(MyCAN_Message_t *received_msg)
{
    /* 在这里处理接收到的CAN消息 */
    switch(received_msg->id) {
        case 0x101:
        	 HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0); // 翻转GPIOA的Pin0电平
            break;
        case 0x201:
            // 例如，更新仪表显示
//            update_display(received_msg->data);
            break;
        default:
            break;
    }
}
```

后面收发的部分和回调函数的注册部分怎么使用看我放在文章最后的github链接即可。

## 踩坑记录

按上面所说搭好CANFD收发代码框架后，我进行了测试。首先我打开了CANFD波特率转换,并且在发送报文的Txheader中也打开了波特率转换，并配置数据段波特率5M，仲裁段波特率1M

```
hfdcan.Init.FrameFormat = FDCAN_FRAME_FD_BRS;
tx_header.BitRateSwitch = FDCAN_BRS_ON;
```

然后连上CAN收发盒并配置好收发盒的相应波特率，然后让MCU每0.5s发送一次报文，发现CAN收发盒的上位机没有读到数据，并且debug中发现CANFD的发送缓冲区满了。

从这些现象中我还是推断不出是什么原因导致的，于是我尝试关闭CANFD波特率转换和Txheader的波特率转换，然后收发盒的数据段波特率配置成1M，发现可以正常实现收发。

然后重要的一步来了，我开启CANFD波特率转换却关闭Txheader的波特率转换，然后发现可以实现收发，并且不管怎么改收发盒的数据段波特率，只要仲裁段波特率为1M，都能实现正常的收发。

这里就产生了很多问题，因为开启CANFD波特率转换只是允许发送波特率可变的报文，此时如果关闭Txheader的波特率转换，本质上就等同于不开启CANFD波特率转换。也就是说发送的报文数据段和仲裁段波特率都是1M（实际上测试用示波器抓取CAN_H的波形并找出数据段中最小周期的高电平发现数据段波特率确实是1M），那为什么收发盒的数据段可以随便改呢？

已知之前测试过收发盒并没有根据总线报文中同步段修改自身数据段波特率这种非常智能的功能，目前我对这个问题的解释就是收发盒在关闭CANFD波特率转换的情况下代码逻辑就是只用仲裁段的波特率而不看数据段的波特率，这也能解释为什么一旦仲裁段波特率改成其它值，就无法实现收发。

在解决了上述问题后我们验证了CAN收发的代码架构是没问题的，但为什么开启Txheader的波特率转换就用不了呢。我在嵌入式论坛上发现一个8月份的帖子里也提到了这个问题，他提到关闭Txheader的波特率转换才能正常使用CANFD,但帖子的作者似乎没有意识到关闭Txheader波特率转换会导致数据段波特率是按仲裁段波特率1M来算而不是自己想要的5M，所以这篇帖子不能作为参考。

然后我尝试看了一下是不是HAL库的问题，简单用VSCODE比对了一下STM32G0和STM32H7（H7之前试过可以使用）的HAL_FDCAN相关部分，发现涉及到BRS的部分几乎相同，于是放弃。

继续逛ST的论坛，里面有提到在如果启用了波特率转换，初始化canfd时最好加上TDC

```
    HAL_FDCAN_ConfigTxDelayCompensation(hfdcan, hfdcan->Init.DataPrescaler * hfdcan->Init.DataTimeSeg1, 0);
    HAL_FDCAN_EnableTxDelayCompensation(hfdcan);
```

没有TDC时，高速数据段的采样点会因延迟而偏移：

- **发送器**：基于本地时钟确定采样点
- **接收器**：收到信号时有延迟，导致实际采样点偏离理想位置

（如果加了这两句代码，cubemx中TX pause那个选项就不用选了）

代码中加了TDC后，我又将CAN线严格双绞了一下，然后配置数据段波特率为2M，发现可以正常收发，而数据段5M的情况下仍无法正常收发，这说明问题就在于CAN通信的链路质量问题了。

我先尝试了数据段4M时修改CAN采样点的配置，发现采样点在75%时可以实现正常通信

然后数据段5M时配置采样点75%发现一条报文都收不到，然后我把CAN信号线剪短，然后重新非常严格地双绞了一遍，发现能发送了，但是发送了6-8条报文后就会停止。

修改同步跳转宽度从2改成5，现在发送10-16条报文然后才会停止

```
hfdcan.Init.NominalSyncJumpWidth = 5;    *// 同步跳转宽度*
```

然后我尝试了不同采样点的情况，然后发现采样点在93.8%时效果最好，经过测试可以实现长时间的稳定通信。这其实是一个很奇怪的现象，正常来说波特率越高，在高速数据段，信号边沿会出现明显的振铃和过冲，如果采样点太晚，可能正好采样在振铃区域，导致误判，更早采样也能为信号稳定留出足够时间。

这是我最后的CANFD相关配置，从理论上讲存在很多问题，但是实际上却是最稳定的一种配置（采样点一设置成13，采样点二设置成2，都无法实现稳定通信）

```
hfdcan.Init.DataTimeSeg1 = 14;   
hfdcan.Init.DataTimeSeg2 = 1;   
hfdcan.Init.DataSyncJumpWidth = 8;  
```

其实到这里已经完成基本功能了，能很稳定地实现5M数据段的收发，但是最后的这个问题我还是无法理解。

> 询问ai得到如下回答：
>
> 虽然最终配置在理论上存在风险（如EMC问题），但在特定短距离场景下确实可行。建议用户后续用示波器观察实际波形，应该能看到93.8%位置恰好是振铃衰减完成的时刻。
>
> ```
> // 您的配置：DataTimeSeg1=14, DataTimeSeg2=1
> // 采样点 = (1+14)/(1+14+1) = 15/16 = 93.75%
> 
> // 物理层真相：
> [信号波形示意图]
> 0-60%：剧烈振铃区 ───╮
> 60-85%：衰减振荡区     │ 传统"危险区"
> 85-93%：临界稳定区     │ 
> 93%-100%：稳定平台 ───╯ ← 您的采样点命中这里
> 
> ```
>
> 所以是终端电阻不匹配导致的？但是60欧和120欧的情况我都试过
>
> 继续询问ai：
>
> ```
> // 在收发器输入端添加并联RC：
> //   R = 33Ω
> //   C = 22pF
> // 可有效抑制振铃（牺牲少量边沿速度）
> ```
>
> 





最后放上开源代码链接：https://github.com/Chesterforever/CANFD_BUTTON
