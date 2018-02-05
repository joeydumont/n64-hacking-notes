---
layout: post
title: DMA on the N64
author: jayd
tags:
 - N64
 - DMA
 - SM64
---

On the N64, there are multiple ways to DMA data from ROM
space to RAM. One is through direct access of the Peripheral
Interface (PI) registers, either in assembly or C, or through
DMA functions within the code of N64 games. 
We detail the three below, choosing the DmaCopy function
from SM64. 


## DMA hook in SM64

In the US version of SM64, the DmaCopy function is located
at `0x80278504` in RAM, andh as the arguments
- DmaCopy(unsigned int RAM_offset, unsigned int, ROM_bottom, unsigned int ROM_top)

