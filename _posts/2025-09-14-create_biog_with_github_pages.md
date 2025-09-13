---
layout: post
title: "QSPI+DMA（接收+发送）的MCU构建实践"
date:   2025-09-14
tags: [芯片通信]
comments: false
author: Lear
---

此处简介

<!-- more -->

## 写在前面

简单略过一下，QSPI即QUAD SPI ，区别于Standard SPI和DUAL SPI的一种六线半双工的SPI数据传输格式，其具体的线序即时序如下图示，相关内容也可以在相关MCU用户手册中查阅。



此次针对C++快速实现的部分，主要已实现普通寄存器的读写功能为主，涉及到的功能码03（0x03）读取保持寄存器值、04（0x04）读取输入寄存器值、06（0x06）写单个保持寄存器

首先是DMA初始化配置

其次是QSPI初始化过程

```
void My_QSPI_Init(void)
{
	spi_parameter_struct spi_init_struct;
	rcu_periph_clock_enable(RCU_AF);
	gpio_pin_remap_config(GPIO_SWJ_SWDPENABLE_REMAP, ENABLE);
	gpio_pin_remap_config(GPIO_SPI0_REMAP, ENABLE);
	rcu_periph_clock_enable(RCU_GPIOA);
	rcu_periph_clock_enable(RCU_GPIOB);
	rcu_periph_clock_enable(RCU_GPIOC);
	rcu_periph_clock_enable(RCU_GPIOD);
	rcu_periph_clock_enable(RCU_SPI0);
	gpio_init(GPIOB, GPIO_MODE_AF_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_3|GPIO_PIN_4|GPIO_PIN_5|GPIO_PIN_6|GPIO_PIN_7);
	gpio_init(GPIOA, GPIO_MODE_OUT_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_15);
	gpio_bit_set(GPIOA,GPIO_PIN_15);
	gpio_init(GPIOD, GPIO_MODE_IPU, GPIO_OSPEED_50MHZ, GPIO_PIN_2);
	gpio_init(GPIOC, GPIO_MODE_OUT_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_12);
	
	spi_init_struct.trans_mode           = SPI_TRANSMODE_FULLDUPLEX;
	spi_init_struct.device_mode          = SPI_MASTER;
	spi_init_struct.frame_size           = SPI_FRAMESIZE_8BIT;
	spi_init_struct.clock_polarity_phase = SPI_CK_PL_HIGH_PH_2EDGE;
	spi_init_struct.nss                  = SPI_NSS_SOFT;
	spi_init_struct.prescale             = SPI_PSC_16;
	spi_init_struct.endian               = SPI_ENDIAN_MSB;
	spi_init(SPI0, &spi_init_struct);
	spi_quad_enable(SPI0);
	spi_quad_io23_output_enable(SPI0);
	spi_enable(SPI0);         
}
```

代码中需要自定义通信接口，设备ID，构建寄存器等，然后在需要的地方对Modbus_RTU函数进行调用即可。
具体逻辑参见下图。
