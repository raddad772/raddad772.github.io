## Emulating in Javascript, and also, emulating the 65c816

And why they're both a "fun" challenge
---

### But first, a note on emulating the 65c816
This thing is kind of a pain. Yes, I know MODR/M gives people nightmares, but those 256 opcodes are composed of roughly 90 instructions applied to roughly 25 unique addressing modes. That number extends to 48 when you take in variants such as Read-Modify-Write.

An example of this is AND, a simple bitwise operation. This instruction is available in a ton of different addressing modes - 15, to be precise. We don't want to write 15 (actually 60, but I'll get to that) versions of the same function, so we want some kind of code that will generically access the data from memory, then call the actual AND algorithm, then complete the instruction. We could then theoretically apply this to other similar instructions like ADC, ORA, and LDA, and save ourselves a lot of code.

Why 120 instead of 15? Because depending on the state of three internal flags in the CPU, and the specific instruction being used, addressing modes perform differently. The Index (X and Y) (X) registers can be toggled between 8- and 16-bits, as can memory accesses and the Accumulator (M), and also, the processor can be in Emulation mode (E), or not. These 3 bit flags (theoretically) make 8 times as many opcodes to emulate. In reality the number is 5, because when E is 1, M and X are also forced to 1.

So, you are either going to have a lot of IF's in your code, which will slow it down a LOT, or you are going to write code to generate the functions that end up executing. This is the approach Higan and JSMoo (my SNES emulator) both take, although we go about it in different ways.

In my case.
