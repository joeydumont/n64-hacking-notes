---
layout: post
title: Progress on the USS64 practice hack
author: jayd
tags:
 - N64
 - DMA
 - SM64
---

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
