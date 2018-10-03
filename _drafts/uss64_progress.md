---
layout: post
title: Progress on the USS64 practice hack
author: jayd
tags:
 - N64
 - romhacking
 - SM64
---

### 2018-10-03

The controller inputs are updated by the game loop at every frame. They are stored in 
[`gPlayer1Controller`]( https://github.com/SM64-TAS-ABC/sm64_source/search?q=gPlayer1Controller&unscoped_q=gPlayer1Controller). The data is stored as a struct:
```
struct Controller
{
  /*0x00*/ s16 rawStickX;       //
  /*0x02*/ s16 rawStickY;       //
  /*0x04*/ float stickX;        // [-64, 64] positive is right
  /*0x08*/ float stickY;        // [-64, 64] positive is up
  /*0x0C*/ float stickMag;      // distance from center [0, 64]
  /*0x10*/ u16 buttonDown;
  /*0x12*/ u16 buttonPressed;
  /*0x14*/ OSContStatus *statusData;
  /*0x18*/ OSContPad *controllerData;
};
```
with the OS structs defined as:
```
typedef struct {
 /*0x00*/ u16     type;                   /* Controller Type */
 /*0x02*/ u8      status;                 /* Controller status */
 /*0x03*/ u8	  errno;
} OSContStatus;

typedef struct {
 /*0x00*/ u16     button;
 /*0x02*/ s8      rawStickX;		/* -80 <= stick_x <= 80 */
 /*0x03*/ s8      rawStickY;		/* -80 <= stick_y <= 80 */
 /*0x04*/ u8	  errno;
} OSContPad;
```
See the [os_cont.h file](https://github.com/SM64-TAS-ABC/sm64_source/blob/c5453afc1574e405b1c5bb7a7faf6d63a19537bc/include/ultra64/os_cont.h)
for the button bitmap.


### 2018-08-18

Apparently, there is a frame advance mode built-in in [SM64](https://hackmd.io/GTbRH81lQo6ajBSDyu138w#Frame-advance).
Could do probably be good for a first version of that feature.

>Frame advance
>This was found by bad_boot. The short at 0x339EC8 is 0 normally and 2 when the game is paused, 4 during transitions (entering a   painting, going through a door, debug level select).>When you hack it to 5 you enter a frame advance mode.

>The game only advances a frame if you press D-pad down then.

### 2018-08-03

The issue was with the `SetCombine` mode of the RDP. I have created a custom combine mode based
on Trevanix and Robin's comments in the Discord. I do not understand why it works, though. It seems
that the mode `G_CC_MODULATEIA` mode does not mesh well with SM64.

Another issue right now is the build process. Before going further, it would be a good idea to have
a working Makefile in order to simplify the build process. Right now the use of `compile.sh` is stupid
and non-scalable. Follow gz's use of static pattern rule to generate rules for the different source files
(the resource files, the uss64 files and the gz files) and have them all in the same `obj` directory.

### 2018-07-28

That's because on the first frame only `gfx_start()` is called, and there is nothing in the DL.
On the second frame we can properly dump the results of the call to `gfx_printf`. However, for the time being,
nothing gets printed on the screen.

I tried changing the calls to `gDPLoadTextureTile` to use physical addresses with
`MIPS_KSEG0_TO_PHYS`, and also the call to `gSPDisplayList`, to no avail. 

Robin gave me his `font{.h,.c}` package to compare. He uses `gSPTextureRectangle` instead
of `gSPScisTextureRectangle`, but I'm not sure that makes a difference. The arguments
they pass to the functions are different though. Setting `s=t=0` in `gSPScisTextureRectangle`
did not help, though.

I have posted in the Discord. I'll take a break from this and wait a little.
Should try to finish the introduction of the thesis before trying to crack the
case of the invisible textures.

### 2018-07-27

It's been a while, and now I have no clue why I get a white screen. Somehow calling `gfx_printf`
seems to be the issue, although I don't understand why.

The issue was that I wasn't linking in `fipps.png.o`. I am a complete idiot. . 

Now the issue is that the RDP seems to terminate the DL immediatley, with the included DL
being simply `00410438:     B8000000 00000000`, i.e. `G_ENDDL`.

### 2018-05-29

The game still hangs after using `MIPS_KSEG0_TO_PHYS(x)` on `gSPDisplayList()`. 
Should go through the display list manually to make sure that it is properly constructed.
Maybe try to put it through SM64Paint, which supposed has a DL editor?

### 2018-05-28

Actually, the problem should be solved by just writing using the `#define MIPS_KSEG0_TO_PHYS(x)     (MIPS_I_(x)&0x1FFFFFFF)`
macro for each reference to a physical address in our DL bulding. Thanks shygoo!


### 2018-05-27

I have an additional problem. The Fast3D microcode docs says that a segmented address
must be given for a display list. Not sure how to put my DL into a segment.
Also not sure how to check that that's what the RSP hangs on.

### 2018-05-24 (cont.)

Turns out the problem was with the optimization flag in `CFLAGS`. I changed it to `-O1`
and `gfx_flush()` had the branch instructions indicative of a loop, like the one gz's
version of `gfx_flush()` does.

However, not I get a white screen at Peach's Letter, presumably because the display list is malformed.
I will try to diagnose that dumping the DL and inspecting it, and also by checking my
`gbi.h` again.

### 2018-05-24

I think I added support for the whole Fast3D microcode to gz's `gbi.h`. However, there is 
nothing that gets printed to the screen. Interestingly, when I compile uss64, the resulting
asm of `gfx_flush()` is very different from gz's. I should try playing around with compile
flags to check whether that is the issue. Specifically, `--gc-sections` could be removing
some [used data](https://stackoverflow.com/questions/31521326/gc-sections-discards-used-data?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa). Although, I don't
know why this flag would remove only parts of a function. Maybe the optimization level?

### 2018-05-14

`GenerateHooks.py` has been added to the build pipeline of the ROM. The ROM runs properly,
but nothing gets printed on the screen. Maybe check with STROOP what's happening at runtime?

TODO:
 - [X] ~~Dump `gfx_disp` and find a way to verify its sanity as a F3D DL.~~ There are still some macros that I still haven't "ported" from F3DZEX to F3D in `gbi.h`.
 - [X] ~~Use STROOP to check whether the DL was inserted properly.~~ I couldn't see my display list, but I'm not sure what's the failure mode of the RDP. Maybe it silently drops a DL it can't parse?

### 2018-05-09

The simplest way to proceed, imo, will be to put all relevant addresses in
`sm64.h`, then use Python/bash to substitute the proper addresses in the armips
script. A nice feature would be to put the ranges and function names of `uss64`
in a `n64split` compatiable format, such that our own code can be diassembled
directly.

TODO:
 - [X] Make a list of the hooks/functions necessary for the armips script.
 - [X] Find a way to intelligently list all the necessary addresses. A text file, yaml file?
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
