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
