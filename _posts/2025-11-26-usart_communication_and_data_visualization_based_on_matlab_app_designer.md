---
layout: post
title: "基于MATLAB App Designer的USART通信与数据可视化"
date:   2025-11-26
tags: [芯片通信]
comments: false
author: Lear
---

如标题所示，这是一个软件模板，基于此模板你可以实现快速开发基于串口通信的上位机，功能基于串口收发，指令按钮，实时数据监视窗口，可以用于常见的上下位机联调等常见场景。

<!-- more -->

## 写在前面

简单写一下用MATLAB APP design实现的一部分优缺点，一是开发便捷易上手，基于UI模块组件以及回调函数逻辑，即使是小白也能借助AI实现零学习成本无痛开发；二是易于数据处理工作；三是自由度低没有办法实现很多自定义，复杂度低只能在模板里实现有限的功能；四是相较于其他开发方式运行环境要求高且运行效率较差。综上，MATLAB上位机非常适用于软硬件开发过程中的调试和测试场景。
```
		function serialconfig(app) % 串口config
            app.SerialPortNumber.Items = serialportlist("all");
            
            if(app.SerialSwitch.Value=='打开')
                delete(app.SerialObject);
                app.SerialState.Color=[0,1,0];      
                
                try
                    app.SerialObject=serialport(app.SerialPortNumber.Value, ...
                        str2double(app.SerialBaud.Value), ...
                        "DataBits",str2double(app.SerialDataBits.Value), ...
                        "StopBits",str2double(app.SerialStopBits.Value), ...
                        "Parity",app.SerialParityBits.Value);
                    %configureTerminator(app.SerialObject,"LF");
                    %configureCallback(app.SerialObject,"terminator",@serialcallback);%不要判断结尾符号
                    configureCallback(app.SerialObject, "byte", 1, @myByteCallback);
                    app.packet.recvBuffer    = [];
                    app.packet.recvBuffSize  = 0;
                    app.packet.Serialcount = 0;
                catch
                    msgbox('串口打开失败');
                    app.SerialState.Color=[0.5,0.5,0.5];
                    app.SerialSwitch.Value='关闭';
                    delete(app.SerialObject);
                end
            elseif(app.SerialSwitch.Value=='关闭')
                delete(app.SerialObject);
                app.SerialState.Color=[0.5,0.5,0.5];
            end
            

            function myByteCallback(src,~)
                % 每次收到字节就重置/启动定时器
                persistent tmr;
                if isempty(tmr) || ~isvalid(tmr)
                    tmr = timer('StartDelay', 0.01, 'TimerFcn', @(~,~) frameComplete(src)); % 10ms 超时
                else
                    stop(tmr);
                    start(tmr);
                     bytesAvailable = app.SerialObject.BytesAvailable;
                    if bytesAvailable > 0
                    newData = fread(app.SerialObject, bytesAvailable, 'uint8');
                    app.dataBuffer = [app.dataBuffer; newData];
                    end
                end
            end

            function frameComplete(src)%接收帧解析
                RecdataDeal(app);
                hexStr = dec2hex(app.dataBuffer, 2);
                hexDisplay = '';
                if(size(hexStr, 1)>4)
                    for i = 1:4
                    hexDisplay = [hexDisplay, hexStr(i, :), ' '];
                    end
                    hexDisplay = [hexDisplay,'...'];
                else
                    for i = 1:size(hexStr, 1)
                    hexDisplay = [hexDisplay, hexStr(i, :), ' '];
                    end
                end
                hexDisplay = ['RX:',hexDisplay];
                app.TextArea.Value = [app.TextArea.Value; hexDisplay];
                TextAreaupdate(app);
                app.dataBuffer = []; 
   
                drawnow limitrate;
            end
        end
```
首先是串口部分，这一部分略过，很多教程里面会讲的比较详细。需要注意的是configureTerminator(app.SerialObject,"LF")这一部分，常规的串口上位机会采用结尾符号来判断一个数据帧的结束，换行符或者是自定义的帧尾，这样的好处是数据接收及时逻辑清晰，但是不够自由。我这里采用的方法是通过定时器和帧间隔判断数据帧的结束，适合数据帧比较清晰的通信过程，不适用于数据量较大或者数据帧粘连的状况，这部分看个人需要。
```
		function RecdataDeal(app)%接收数据处理
            if(app.ButtonDataRec.Value==1)%绘图
                
                if (app.dataBuffer(3)==5)
                    data = app.dataBuffer(5)*256*256+app.dataBuffer(6)*256+app.dataBuffer(7);
                    if isempty(app.axes_dataX)
                        app.axes_dataX = 1; 
                    else
                        newIndex = app.axes_dataX(end) + 1;
                        app.axes_dataX = [app.axes_dataX, newIndex];
                    end
                    app.axes_dataY = [app.axes_dataY, data];
                    if length(app.axes_dataY) > 100
                        app.axes_dataX = app.axes_dataX(2:end);  
                        app.axes_dataY = app.axes_dataY(2:end);  
                    end
                    plot(app.UIAxes, app.axes_dataX, app.axes_dataY);

                end
                
             end
        end
```
其次是数据接收后的处理，做一些简单的校验，提取自己的数据输出到UIAxes图表当中，看起来不是很美观，所以还有很多可以优化的空间。
```
        function TextAreaupdate(app)%更新文本框
             maxLines = 40;
             if numel(app.TextArea.Value) > maxLines
                 app.TextArea.Value = app.TextArea.Value(end - maxLines + 1:end);
             end
             app.TextArea.scroll('bottom');
        end
```
发出的数据帧和接收帧都会处理后打印到文本框中起到提示debug的功能
```
        function ButtonAck(app,data)
            data = [0xAA, 0x02, 0x05,0xB1];   % 测距
            SerialSend(app,data);
        end
```
按钮添加注入此类的回调函数即可
