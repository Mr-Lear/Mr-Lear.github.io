---
layout: post
title: "TEST"
date:   2025-08-27
tags: [TTT]
comments: true
author: Lear
---

AAA

BBB

<!-- more -->

## 开始
```
#define TX_LEN 			99
#define SLAVE_ID 		50	//1-247
#define Modbus_Send 	My_Transmit_fun  //My_Transmit_fun(int8_t *data,uint len)

typedef struct {
    uint16_t 	reg0; 
} Modbus_InputREG;

typedef struct {
	uint16_t 	reg0;
} Modbus_HoldingREG;

Modbus_HoldingREG modbus_reg3 = {0};
Modbus_InputREG modbus_reg4 = {0};

uint16_t CRC_Check(uint8_t *data, uint16_t len)
{
	uint16_t crc = 0xFFFF;
	while (len--) 
	{
        crc ^= *(data++);
        for (int i = 0; i < 8; i++) 
		{
            if (crc & 0x0001) 
			{
                crc = (crc >> 1) ^ 0xA001;
            } 
			else 
			{
                crc >>= 1; 
            }
        }
	}
	return crc;
}

void Endian_X16(uint8_t *data, uint8_t len)
{
	uint8_t data_t = 0;
	for(int i=0;i<len/2;i++)
	{
		data_t = data[i*2];
		data[i*2] = data[i*2+1];
		data[i*2+1] = data_t;
	}
}

void Modbus_RTU(uint8_t *RX_buffer, uint8_t RX_len)
{
	int8_t 		tx_t[TX_LEN];
	uint8_t 	exceptionCode = 0;
	uint8_t 	address = RX_buffer[0];
	uint8_t 	function = RX_buffer[1];
	uint16_t 	start = (RX_buffer[2]<<8|RX_buffer[3]);
	uint16_t 	cnt_t = (RX_buffer[4]<<8|RX_buffer[5]);
	uint16_t 	crc_t = CRC_Check(revcbuffer,rec_len-2);
	
	if(address==SLAVE_ID && (RX_buffer[RX_len-1]==(crc_t>>8) && (RX_buffer[RX_len-2]==(uint8_t)crc_t))
	{
		tx_t[0] = address;		
		tx_t[1] = function;
		switch (function)
		{
			case 0x03:    //Read Holding Registers
				if(cnt_t>0x7D || cnt_t<0)
				{	exceptionCode = 0x03;	break;}		//Out of range(0x7D)
				if((cnt_t<=(sizeof(modbus_reg3)>>1)) && (cnt_t+start <=(sizeof(modbus_reg3)>>1)))
				{
					tx_t[2] = cnt_t * 2;
					__disable_irq();
					memcpy(tx_t+3,(uint8_t *)&modbus_reg3+start *2,cnt_t * 2);
					__enable_irq();
					uint16_t crc = CRC_Check(tx_t,cnt_t * 2 +3 );
					memcpy(tx_t+cnt_t*2 + 3,(uint8_t *)&crc,2);
					delay_1ms(1);
					if(SET != usart_flag_get(USART0, USART_FLAG_IDLE))
					{	Modbus_Send(tx_t,cnt_t*2 +5);}
				}
				else
				{ 	exceptionCode = 0x02;}    //Out of range(size of reg)
				break;
			case 0x04:    //Read Input Registers
				if(cnt_t>0x7D || cnt_t<0)
				{	exceptionCode = 0x03;	break;}		//Out of range(0x7D)
				if((cnt_t<=(sizeof(modbus_reg4)>>1)) && (cnt_t+start <=(sizeof(modbus_reg4)>>1)))
				{
					tx_t[2] = cnt_t * 2;
					__disable_irq();
					memcpy(tx_t+3,(uint8_t *)&modbus_reg4+start *2,cnt_t * 2);
					__enable_irq();
					uint16_t crc = CRC_Check(tx_t,cnt_t * 2 +3 );
					memcpy(tx_t+cnt_t*2 + 3,(uint8_t *)&crc,2);
					delay_1ms(1);
					if(SET != usart_flag_get(USART0, USART_FLAG_IDLE))
					{	Modbus_Send(tx_t,cnt_t*2 +5);}	
				}
				else
				{ 	exceptionCode = 0x02;}
				break;		
			case 0x06:    //Write Single Register
				memcpy(tx_t,RX_buffer,8);
				uint8_t addr_t = RX_buffer[3];
				if(addr_t<(sizeof(modbus_reg3)>>1))
				{
					__disable_irq();
					memcpy((uint8_t *)&modbus_reg3,RX_buffer+5,2);
					__enable_irq();
					Modbus_Send(RX_buffer, 8);
				}
				else{	exceptionCode = 0x04;}
				break;
			default:
				exceptionCode = 0x01;    //function not supported
				break;
		}
		if(exceptionCode)
		{
			tx_t[1] = 0x80|tx_t[1];
			tx_t[2] = exceptionCode;
			uint16_t crc = CRC_Check(tx_t,3);
			tx_t[3] = crc;tx_t[4] = crc>>8;
			Modbus_Send(tx_t, 5);
		}
	}
}
```
