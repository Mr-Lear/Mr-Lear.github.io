---
layout: post
title: "基于开源Modbus库的MCU工程搭建"
date:   2025-08-28
tags: [工业通信]
comments: false
author: Lear
---

当我们需要更加深入的Modbus功能而不只是局限于简单实现的情况下，就可以考虑调用成熟的Modbus库来实现我们的需求了，不仅包含了全面的Modbus功能码，还能实现接收列队的FIFOF，不丢失每一个应答帧，能够从容地应对更加复杂的应用场景

<!-- more -->

## 写在前面

