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
We detail the three below.


## DMA hook in SM64

Version| Address
-------|--------
SM64-U | `0x80278504`
SM64-J | ``
SM64-S | ``


The DmaCopy function has the following signature:
```C 
DmaCopy(unsigned int RAM_offset, unsigned int, ROM_bottom, unsigned int ROM_top)
```

