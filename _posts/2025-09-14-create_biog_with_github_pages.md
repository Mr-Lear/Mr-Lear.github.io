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

首先是DMA初始化配置，配置流程与常规SPI+DMA配置无异，可参考芯片给定例程，绑定通道需要查阅用户手册确认，依据部分提示DMA配置需要先于SPI初始化。
```
void SPI_DMA_Init(void)   
{
	rcu_periph_clock_enable(RCU_DMA0);  
    dma_parameter_struct dma_init_struct;
	
    dma_deinit(DMA0, DMA_CH1);
    dma_init_struct.periph_addr  = (uint32_t)(&SPI_DATA(SPI0));
    dma_init_struct.memory_addr  = (uint32_t)rec_data;
    dma_init_struct.direction    = DMA_PERIPHERAL_TO_MEMORY;
    dma_init_struct.memory_width = DMA_MEMORY_WIDTH_8BIT;
    dma_init_struct.periph_width = DMA_PERIPHERAL_WIDTH_8BIT;
    dma_init_struct.priority     = DMA_PRIORITY_HIGH;
    dma_init_struct.number       = 12;
    dma_init_struct.periph_inc   = DMA_PERIPH_INCREASE_DISABLE;
    dma_init_struct.memory_inc   = DMA_MEMORY_INCREASE_ENABLE;
    dma_init(DMA0, DMA_CH1, &dma_init_struct);
    dma_circulation_disable(DMA0, DMA_CH1);
    dma_memory_to_memory_disable(DMA0, DMA_CH1);
	dma_channel_disable(DMA0, DMA_CH1);
	
	dma_deinit(DMA0, DMA_CH2);
    dma_init_struct.periph_addr  = (uint32_t)&SPI_DATA(SPI0);
    dma_init_struct.memory_addr  = (uint32_t)tx_data;
    dma_init_struct.direction    = DMA_MEMORY_TO_PERIPHERAL;
    dma_init_struct.memory_width = DMA_MEMORY_WIDTH_8BIT;
    dma_init_struct.periph_width = DMA_PERIPHERAL_WIDTH_8BIT;
    dma_init_struct.priority     = DMA_PRIORITY_LOW;
    dma_init_struct.number       = 12;
    dma_init_struct.periph_inc   = DMA_PERIPH_INCREASE_DISABLE;
    dma_init_struct.memory_inc   = DMA_MEMORY_INCREASE_ENABLE;
    dma_init(DMA0, DMA_CH2, &dma_init_struct);
    dma_circulation_disable(DMA0, DMA_CH2);
    dma_memory_to_memory_disable(DMA0, DMA_CH2);
}
```
其次是QSPI初始化过程，与常规SPI配置类似，需要个人确认分频，极性等相关内容，部分区别于SPI，参考芯片手册内容QSPI仅支持Master模式，需要spi_quad_enable和spi_quad_io23_output_enable类似的使能配置

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

参考常规QSPI的硬件读写时序，由于其半双工特性区别于常规双工SPI，基于之前对单线半双工的SPI 软件通信协议的实现过程的了解，需要在每次READ工作前将DATA线上拉至高电平的一个动作，在常规读写流程中以SPI_DATA(SPI0) = 0xFF表现，另外需要定义QSPI读写方向，因此考虑到DMAREAD过程中也需要类似的SPI_DATA(SPI0) = 0xFF动作，所以想到可以加入同步进行的DMA写入FF流，经验证终于可以实现连续的DMAREAD数据流，至此DMA+QSPI无法DMA读出的问题解决。

