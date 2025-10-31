---
title: DJI robomaster C板陀螺仪驱动调试寄录
date: 2023-011-08 23:48:54
tags: [robomaster,IMU]
index_img: /img/avatar.png

---

# robomaster C板陀螺仪驱动调试寄录

首先robomaster官方C板用户手册有关于陀螺仪的使用说明和例程，但是细看会发现很多代码只是勉强能跑，只能用于学习，想要有良好的性能实现比赛场上的高强度运行，必须重新编写自己的驱动。队内上一届表现最好的全向移植的是南航开源的IMU驱动版本，但是我对其代码风格不太感冒，所以打算自己移植BMI088官方的驱动库然后再加上自己从南航的版本中学到的一些小技巧。从结果上看，我只能说相比于MPU6050，BMI088在性能上完爆但是开发体验并不是很好

## 移植步骤

首先根据README给出的指引，我们只需要编写读写SPI的函数与微秒延时函数即可。微秒级延时可以直接用Systick当计数器完成（后面了解到其实最好用DWT），而SPI通信的移植就比较复杂了

### SPI通信

C板参考例程内CubeMX的SPI速率配置为300k，根据测试和计算，这个速率肯定是太慢了。按照每次只读取6个字节（三轴加速度计与三轴角速度计数据都是2Byte，实际只会更多）共48bit来算，至少0.2ms的时间耗费在SPI通信上。但是根据官方数据手册上给出的SPI参数，最大速率能达到到10Mhz，再根据在[Application Note](https://www.bosch-sensortec.com/media/boschsensortec/downloads/application_notes_1/bst-mis-an006.pdf)内提到推荐2Mhz以上的通信速率。最终通过计算应用层的数据链路传输频率，我将SPI速率配置为2.625Mbps，后续经过测试，基本能实现生产和消费之间的平衡（其实不需要这么精确，只是我对于性能有强迫症）

{% asset_img spi-speed.png  %}

接下来就是软件方面上我从南航的代码中学到的小技巧了，SPI的读写如果使用HAL库的spi函数进行一次传输，传输速度是慢于直接用寄存器完成传输的，对于像BMI088这样对实时性要求极强的IMU，应该直接对寄存器操作来减小开销

```
uint8_t spi_rw_byte(uint8_t byte){
	SET_BIT(BMI088_SPI->CR1, SPI_CR1_SPE);
    while((BMI088_SPI->SR & SPI_SR_TXE) == RESET);
	BMI088_SPI->DR = byte;
    while((BMI088_SPI->SR & SPI_SR_RXNE) == RESET);
    return BMI088_SPI->DR;
}
```

IMU加速度计与角速度计在同一个SPI总线上通过不同CS引脚选中，对应到BMI088官方库的移植函数则是通过传入的`intf_ptr`指明，其值是初始化中的`intf_ptr_accel`或`intf_ptr_gyro`，这里只需要判断传入参数拉低对应的片选脚即可

```
BMI08X_INTF_RET_TYPE bmi08x_spi_read(uint8_t reg_addr, uint8_t *reg_data, uint32_t len, void *intf_ptr);
BMI08X_INTF_RET_TYPE bmi08x_spi_write(uint8_t reg_addr, const uint8_t *reg_data, uint32_t len, void *intf_ptr);
```

还有一个我遇到的SPI通信中比较特殊的问题是，在每次与**加速度计**通信时不论是读单个寄存器还是连续读都需要先接收一个Dummy Byte，这个东西实际上是没有意义的内容。DJI C板手册中似乎没有写对这个问题的处理，但是看了一下BMI088官方的驱动是有相关的处理方式的

{% asset_img spi-comm.png  %}

### 初始化流程

我认为不管哪款IMU，初始化都是最难且最有可能出问题的那一步。作为MEMS器件，在上电之后需要静置稳定，不论是等待加速度计的数据稳定还是角速度计元件的起振，并且BMI088没有任何办法告诉主机它是否已经完全就绪。

通过手册得知，IMU内部也有电源管理，其数字接口与模拟部分可以分别供电且电源模式分为Suspend Mode和Normal Mode，电源模式之间的切换也需要等待稳定。了解这个是非常重要的，因为从寄存器的初始值来看，BMI088经过上电复位后加速度计是处于Suspend Mode，而角速度计是处于Normal Mode

总之按照数据手册的说法和我实际不加延时的测试，BMI088的初始化的等待至少需要

- 加速度计上电等待1ms，切换电源模式进入Normal Mode后等待50ms
- 角速度计上电或切换电源模式都需要等待30ms
- SPI两次写入间至少需要有2us

于是我们可以分析一下，一个完整的初始化流程应该是这样的

- 上电复位，等待30ms至IMU完全就绪
- 分别切换角速度计和加速度计的电源模式，每次切换等待30ms或50ms
- 写入输出速率，量程大小等配置，每次写入需要确保间隔2us以上（写间隔可在移植的SPI读写函数内完成）
- 等待数据采集

其实在各种电源模式下读写寄存器的时序连官方自己都搞不太清，在移植驱动的过程中官方的仓库一直在更新，但是好在目前我移植过程还算顺利

（2024.6.18更新）我发现了一个很有意思的点，博世的官方驱动在初始化程序里给BM088上传一个类似通过软件更新固件一样的包，通过调用`bmi08a_load_config_file`函数。我在逛博世论坛时发现有人提出了这个问题并做了解释，这个东西没什么用处除非你要使用数据同步的特性（这个功能C板没戏，硬件连线都没有引出来），所以最好不要加这个以免出现不必要的初始化错误。

最后放一下我的参考资料，希望我的踩坑寄录能帮助到后续开发的同学

[C板官方例程](https://github.com/RoboMaster/DevelopmentBoard-Examples)

[BoschSensortec](https://github.com/BoschSensortec)/[BMI08x-Sensor-API](https://github.com/BoschSensortec/BMI08x-Sensor-API)

[BMI088 DataSheet](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmi088-ds001.pdf)