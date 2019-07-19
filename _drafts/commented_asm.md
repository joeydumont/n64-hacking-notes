```mipsasm
// Level Reset (OG asm)
lui   t0, 0x8034
lui   t1, 0x8036
lb    at, 0x9C31 (t0) // Load lower 8 bit of buttonDown
addiu v0, r0, 0x0020  // Load L button mask.
bne   at, v0, LvlResetEnd // If L is not pressed, skip.
nop
addiu v0, r0, 0x0880 // Load v0 with 0x0880
sh    v0, 0x9EAE (t0) // Fills Mario's health
sh    r0, 0x9EF2 (t0) // Sets displayed coins to 0
sh    r0, 0x9EA8 (t0) // Sets number of coins in Mario state to 0.
addiu v0, r0, 0x0002
sb    v0, 0x9ED8 (t0) // Sets sCurrWarpType to 0x0002.
addiu v0, r0, 0x0005
sh    v0, 0x00A4 (t1) // Sets snow particle count to 5 (???)
LvlResetEnd:

// Level Reset Camera Fix
lui   t0, 0x8028
lui   t1, 0x8034
lb    at, 0x9ED9 (t1) // Loads destination level ID
addiu v0, r0, 0x001D
beq   at, v0, TotWC // Checks that level is TotWC
nop
Fix:
  addiu v0, r0, 0x0001
  sh    v0, 0x6D2A (t0) // I think that rewrites some code??? 80286D28 is lbu   $t6, ($t5)
TotWC:
  bne at, v0, EndFix // If level is not TotWC, skip to the end.
  nop
  sh r0, 0x6D2A (t0) // Zeros some code instead of rewriting.
EndFix:

jr ra
nop
```
