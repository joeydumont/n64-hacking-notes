---
layout: post
title: Progress on the USS64 practice hack
author: jayd
tags:
 - N64
 - DMA
 - SM64
---

### 2018-03-21
Next steps: 
  - [ ] verify, with PJ64d, that calling `gfx_printf()` with F3DEX2 is indeed the cause of the infinite loop.
  - [ ] If the above is true, add Fast3D support to glank's `gbi.h`.
  - [ ] Find a display list in SM64's RAM and try to determine how to access it.

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
