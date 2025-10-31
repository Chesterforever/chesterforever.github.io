---
title: 有效减少莫名奇妙的bug的一些方法
date: 2025-08-31 01:03:24
tags: [MCU,C]
index_img: /img/avatar.png

---

# 有效减少莫名奇妙的bug的一些方法

```
// 禁用DMA
__HAL_DMA_DISABLE(&hdma_spi1_rx);
// 等待DMA被禁用（确保DMA已停止）
while(hdma_spi1_rx.Instance->CR & DMA_SxCR_EN)
{
    __HAL_DMA_DISABLE(&hdma_spi1_rx);
}
```

- 禁用DMA后加上判断确保DMA已停止
- 直接使用HAL_DMA_Abort似乎能代替这一过程（待验证）



    uint8_t ist8310_init(void)
    {
        static const uint8_t wait_time = 1;
        static const uint8_t sleepTime = 50;
        uint8_t res = 0;
        uint8_t writeNum = 0;
        ist8310_GPIO_init();
        ist8310_com_init();
    
        ist8310_RST_L();
        ist8310_delay_ms(sleepTime);
        ist8310_RST_H();
        ist8310_delay_ms(sleepTime);
        
        //下面为主要初始化代码
        ......
    }

- 在对有硬件电路的对象（如电机，传感器）初始化或复位时调用复位函数后要进行延时等待硬件电路初始化





