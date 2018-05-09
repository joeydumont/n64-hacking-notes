---
layout: post
title: Progress on the USS64 practice hack
author: jayd
tags:
 - N64
 - romhacking
 - SM64
---

### 2018-05-09

The simplest way to proceed, imo, will be to put all relevant addresses in
`sm64.h`, then use Python/bash to substitute the proper addresses in the armips
script. A nice feature would be to put the ranges and function names of `uss64`
in a `n64split` compatiable format, such that our own code can be diassembled
directly.

TODO:
 - [ ] Make a list of the hooks/functions necessary for the armips script.
 - [ ] Find a way to intelligently list all the necessary addresses. A text file, yaml file?
 - [ ] Read RAM entry point from ROM?

### 2018-05-03

After some help from shygoo and camthesaxman, it seems that doing 
`#define gDisplayListHead (*(Gfx **)0x8033B06C/0x80339CFC)` and calling
`gSPDisplayList(gDisplayListHead++)` should do the trick. 

The trick is to hook at `0x80247D1C` into `CleanupDisplayList`, which is called at the
end of every frame. To do that in C, it will be necessary to insert a hook in there by
patching the ROM with a jump to a function in uss64. To do that, we'll be inspired by
gz's genhooks function. Our hook will probably need to call the overwritten function
by using asm, as calling it in C put the game in an infinite loop.

Here's the asm example by shygoo:
```asm
; 80247D14 function writes final rdpfullsync (E9) and enddl (B8) to the master dl (called once every frame)
; 8033B06C master dl tail pointer

; example procedure for pushing commands to the end of the master dl

.org 0x80247D1C
    jal 0x80370000
    
.org 0x80370000
    addiu sp, sp, -0x18
    sw    ra, 0x14 (sp)
    jal   0x8024784C ; do function call we replaced
    nop
    lui   t0, 0x8034
    lw    t1, 0xB06C (t0) ; t1 = dl tail pointer
    
    li    t2, 0xDEADBEEF
    sw    r0, 0x00 (t1)
    sw    t2, 0x04 (t1) ; write 00000000 DEADBEEF (g_noop) to dl
    addiu t1, t1, 8
    
    sw    t1, 0xB06C (t0) ; increment tail pointer by num bytes we pushed
    lw    ra, 0x14 (sp)
    jr    ra
    addiu sp, sp, 0x18
```
And another example using a fillrect:
```asm
    li t2, 0xF7000000
    li t3, 0x0000F0FF
    sw t2, 0x00 (t1)
    sw t3, 0x04 (t1)
    addiu t1, t1, 8
    
    li t2, 0xF60E00E0
    li t3, 0x00060060
    sw t2, 0x00 (t1)
    sw t3, 0x04 (t1)
    addiu t1, t1, 8
```

### 2018-04-24

Actually, this last pointer can be found in the RDP initialization function, called `myRdpInit` in n64split. 
In the US version, we would have `gDisplayListHead = 0x8033B06C`. Not sure if it's wise to mess with the master
display list in this way. A good idea would be to monitor the calls to `*alloc_displaylist(args) = 0x8019CF44`
and hook into one of the display lists created.

### 2018-04-23
Not sure how to find a pointer to a display list. Maybe try with the Japanese display list at `gDisplayListHead = 0x80339CFC`.
Would have to refactor uss64 to accept the J rom though.

### 2018-04-09
I have determined that the `_SHIFTL(v,s,w)` macro is equivalent to glank's `gF_(v,w,s)` macro.
Notice the different argument order.

Instead of hijacking an existing SM64 display list, it might be easier, in the long run, to use
our own display list. SM64's has two functions that might be of use
 - `[0x8019CF44, "alloc_displaylist"]`
 - `[0x80246C10, "SendDisplayList"]`
 
Their arguments are not documented though, so I should check the diassembly and figure that out.

### 2018-03-29

It seems that `_SHIFTL` and `_SHIFTR` are defined in [SGI's MBI](http://n64devkit.square7.ch/header/mbi.htm) to be
```C
#define _SHIFTL(v, s, w)	\
    ((unsigned int) (((unsigned int)(v) & ((0x01 << (w)) - 1)) << (s)))
#define _SHIFTR(v, s, w)	\
    ((unsigned int)(((unsigned int)(v) >> (s)) & ((0x01 << (w)) - 1)))
```

### 2018-03-21
Next steps: 
  - [X] verify, with PJ64d, that calling `gfx_printf()` with F3DEX2 is indeed the cause of the infinite loop.
  - [X] If the above is true, add Fast3D support to glank's `gbi.h`.
  - [X] Find a display list in SM64's RAM and try to determine how to access it.

### 2018-03-18
I am an idiot. The issue was not the VMA of the binary file, but rather the
fact that the use of glank's `gfx_printf()` made the game hang. I have not
isolated the specific issue yet, but glank's `gbi.h` only supports F3DEX2,
while SM64 uses the much earlier Fast3D RSP microcode. To make everything work,
it is highly probable that I will need to either rewrite `gbi.h` with Fast3D in
mind, or simply reuse SGI's `gbi.h`, with modifications.


### 2018-02-17
Directly writing the address in the behaviour script, i.e. `.dw 0x80400000`
works when we use the assembled object file `hello_world.o` via `.importobj`.
However, manually linking `hello_world.o`, even when specifying the absolute
address of `MainHook` as `0x80400000`a via `-Wl,--defsym,start=0x80400000`,
where `start` is defined in glank's custom linker script, results in issues
when running the ROM (mupen seems to show an infinite loop, PJ64d pops up
an "address error" dialog.

Future steps: take a look at [queueRAM's SM64 specific linker script](1) to understand
how to properly do this.

[1]:https://github.com/queueRAM/sm64tools/blob/02bf6273d07c1c4304626b2b6f6ba44a3dc63ca4/examples/hello_c/hello_c.patch

### 2018-02-14

Use of the `gz` codebase seems incompatible with the use of `importobj` in armips.
A lot of `gz`'s functionality relies on the custom linker script `gl-n64.ld`, which
defines the sections of the produced binary file. I'm not sure how to instruct `armips`
with custom linker options.

However, `importobj` allowed direct use of the symbols defined in the object file.
As such, it was possible to write `.dw MainHook` to insert a function pointer
in the Mario behaviour script. By linking the payload with the custom linker
script beforehand, we must instead include a binary file in the ROM, with
`incbin`. This precludes the use of `.dw MainHook`, as the binary import
is blind to the symbols. Trying to directly write the function address 
`.dw 80400000` fails, resulting in an infinite loop.

Should either try to customize the armips linker or figure out how to properly
insert a function pointer to my code using the `incbin` binary import.
