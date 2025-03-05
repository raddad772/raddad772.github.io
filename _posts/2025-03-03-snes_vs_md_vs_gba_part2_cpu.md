## Super Nintendo vs. Genesis/MegaDrive vs. GameBoy Advance, part 2: CPU

I've decided to order these in order of power, though as we'll see, this category isn't totally easy to call any specific way.

We'll also discuss the memory subsystems and cartridges of the systems, since these affect the speeds these CPUs actually process.

Remember, the most important metric is really how much data can be processed how fast. 

### Basic specs

* SNES:
  * Main CPU: Western Design Center 65816
    * 1.79-3.58MHz, varying in real-time depending on the job it's doing.
    * 24-bit address bus, 8-bit data bus
    * Base instruction size is 8 bits, with optional operands of up to 3 more bytes IIRC
    * Memory access in 1 cycle
    * Max 3.58 MNPS (Million NOPs Per Second), the fastest instructions can move through the processor.
    * Multiply/divide:
      * Unsigned 8x8-bit = 16-bit multiply in 8 cycles
      * Unsigned 16bit divided by 8bit = 16-bit result, 16-bit remainder: 12-13 cycles
      * Signed 8x8-bit = 16x8-bit = 24-bit multiply in 1 cycle, via the PPU. This could not be used outside of vblank if mode 7 was rendering. 
  * Sound CPU will be handled in the APU section...
* Genesis:
  * Main CPU: Motorola 68000
    * 7.6Mhz
    * 24-bit address bus, 16-bit data bus
    * Base instruction size is 16 bits (1 native "word" is 16 bits), with optionally some 16-bit operands
    * Memory access in 4 cycles
    * Max 1.9 MNPS (Million NOPS Per Second), because each memory access takes 4 cycles.
    * Multiply/divide:
      * These instructions took varying amounts of time depending on the operands. n below refers to the number of 1's in one operand...
        * Unsigned 16x16-bit to 32-bit multiply: 38 + 0-32
        * Signed 16x16-bit to 32-bit multiply: 38 + 0-18
        * Signed/unsigned 32/16-bit to 32-bit divide: 140-150 cycles!  
  * Sound CPU will be handled in the APU section...
* GameBoy Advance:
  * Main CPU: ARM ARM7TDMI (we're ignoring the GBC hardware since it mostly can't be used during GBA games)
    * 16.67MHz
    * 32-bit address bus, 32-bit data bus
    * Base instruction size is 16 or 32 bits (1 native "word" is 32 bits), with no operands, except for PC-relative addressing, which is sort of like an operand
    * Memory access in 1 cycle
    * Max 16.6 MNPS, but don't let this number fool you, actual performance was very often MUCH LESS....
    * Multiply/divide:
      * 32x32=32-bit multiply = only 2-5 cycles!
      * No hardware for division! The compiler provides routines that can be <45 cycles though. I'm not sure on exact notes.

### WDC 65816 (SNES)
At first glance, these specs look pretty OK. Its maximum is double that of the processor in the NES. But when we look deeper, a lot of issues crop up.

First and foremost, this CPU is an extension to the 8-bit CPU used in the NES, the MOS Technologies 6502. This is a huge blow to it already. The processor features only one general-purpose register and just a few indexing/addressing registers, relying instead on external memory during intensive calculations. It also used the same ALU for general-purpose processing as when doing addressing calculations, which meant that it could not do these in parallel. This worked fine OK on 8-bit systems, and would work fine here too, except for the next flaw...

Although it has a 24-bit address bus, it really only works on 64K "chunks" of memory at a time. So if you want to copy data between 64K chunks of memory, or move around and work on lots of separate areas, you need more instructions to change which "bank" of memory you're working on.

The SNES designers mitigated this somewhat by making an overcomplicated memory mapping scheme that put all of the hardware IO, as well as 8KB of RAM and 32KB slices of ROM, into many 64k chunks. When you wanted to, say, load lots of data, you were supposed to load the program that would do so into the 8KB of RAM (aka lowRAM), and then execute out of there, copying chunks into lowRAM and then into where you wanted them in high RAM, or just copying directly with some speed loss.

Since you also didn't have a lot of registers in the CPU, this meant that sometimes there would be a lot of cycles just wasted changing which chunk of RAM you're working in. It was possible to write very well-optimized code and get around many of these limitations, but it wasn't simple or easy.

The next challenge this processor faces is that it very rarely actually ran at 3.58MHz. It mostly ran at 2.7MHz, when accessing RAM, or ROM. Or, if the cart used particularly slow ROM, it wuld run at 1.79Mhz (same as the NES) while reading from it.

Finally, it only has an 8-bit data bus. To operate on 16-bit data, it must issue two 8-bit loads, operate on it internally, then 2 8-bit stores to write the data back. And since there weren't even a lot of registers inside to hold the data, this had to be done a lot.

You could optimize a good deal by using 8bit data wherever possible, but this did place more limitations.

Ultimately, although this could execute more MIPS than the Sega Genesis, unless you coded VERY skillfully, you were likely to get a lot less throughput.

Finally, for more advanced multiplications, the CPU was actually relatively speedy, at least compared to the M68000. See the numbers above. However, this is reduced by the amount of time it actually took to read the operands, write to the multiplier units, read back an answer, and write them back to memory.

So we end up with an interesting processor that you CAN squeeze some performance out of, which, however, has MANY challenges in practice that slow it way down.

### Motorola 68000 (Genesis)
The M68000 is an interesting design. Internally, its registers are 32-bit, but the ALU that actually does mathematical operations is 16-bit. This means that theoretically, a 32-bit ADD should take twice as long as a 16-bit ADD. However, the designers were clever, and often could hide this. And almost all operations could be done in 16-bit mode, and many in 8-bit mode. Most programmers exclusively used 8- and 16-bit variants of instructions, unless 32 bits were needed for some reason (and 32-bit copies could be faster than 16-bit ones too).

The M68000 had dedicated Effective Address calculation units, which sped up calculation of addresses, and in some cases this could be done in parallel with some data processing. It had many flexible addressing modes that made developing on it fairly simple and nice.

Unlike the SNES's 65816 which does memory accesses in 1 CPU cycle, the M68000 requires 4 for each (or 5 for a test-and-set which doesn't work on the Genesis anyway so we won't worry about it). It also uses a varying number of internal cycles, those these almost always come in multiples of 2 for various reasons. Reads and writes are all 16 bits, so reading a byte requires the same time as reading 16 bits. Reading 32 bits requires two memory cycles, aka 8 machine cycles.

This means that, although it can only do 1.9 million reads or writes per second MAX as compared to the SNES 65816's 2.7-3.5, it transfers twice as much in each one. Except for when you really need to access only 8 bits at a time, this is a speed advantage to the M68K.

And due to the generous amounts of internal registers and the address calculation units, it makes much better use of the bandwidth it has, with MANY less intermediate instructions required during calculations. 

It also has more addressing modes, and 8 internal registers dedicated to holding 32-bit addresses. This means that calculations on which parts of memory to operate on are often more convenient and faster than the 65816.

Also, it has 8 general-purpose 32-bit data registers, as opposed the the 65816's single 16-bit (cut in half as two 8-bit ones in some cases). This means it didn't need to access memory nearly as often when doing mathematical operations.

Overall this means that while the M68000 may have had roughly similar theoretical bandwidth as the 65816, it generally made MUCH better use of it, leading to faster overall calculations.

### ARM7TDMI (GBA)
On paper, this thing is a beast. Compared to the M68000's 8 general-purpose and 8 address registers, this has 16 general-purpose registers (one of which is dedicated to the Program Counter, but can still be used in many operations), which can be used for either use.

Compared to taking 4 cycles per memory access, the ARM7TDMI itself is capable of doing memory accesses in only one cycle.

And it runs at over twice the clock rate!

So why does Sonic the Hedgehog Genesis, a port of a Sega Genesis game, run so BADLY? But conversely, why are 3d games at all even POSSIBLE?

Well, how fast this processor ACTUALLY is and its ACTUAL throughput depends on a lot. Nintendo made a lot of decisions in the GBA that really crippled its theoretical performance. The CPU spends a LOT of its time just waiting around on memory accesses.

First and foremost, cartridges are SLOW. An access to a cartridge can take 8 cycles. And the bus to them is only 24-bit TOTAL for both address AND data, so to get a whole 32-bit instruction out of it, it can THEORETICALLY take 10 cycles.

Nintendo was sort of smart about engineering this, and consecutive accesses are faster than random ones. The hardware also includes a pre-fetcher which will make executing code from the cart quite a bit faster if it is structured the right way. Finally, the ARM7TDMI has a second mode called "THUMB" which has a native 16-bit instruction size. They're less flexible with less access to registers (typically only 8 + PC instead of 16), but only need 16-bit accesses for each instruction. This is supposed to run about 60% faster from the cartridge overall than 32-bit mode.

As for the rest of the system, the BIOS, 32K of work RAM, and the MMIO registers (like for communicating with the PPU) are all single-cycle 32-bit access. However the 256K of RAM and VRAM are 16-bit busses (meaning two accesses are necessary for a 32-bit access). VRAM is a single cycle, but the 256K of work RAM is 3 cycles per 16-bit access.

So the only time you'd generally get FULL performance out of the processor, would be when working inside the 32K of fast RAM, perhaps while drawing to VRAM or doing maths.

Further, you get a little less out of each instruction. The 65816 and 68000 are CISC, meaning Complex Instruction-Set Computer, in philosophy. You could do all sorts of complicated addressing modes to access data, encoded as part of the instruction.

The ARM7TDMI is RISC in philosophy. This means Reduced Instruction Set Computer. The idea was to enable faster speeds and advanced techniques like pipelining and instruction re-ordering by simplifying instructions. So although a 16-bit load may take 3-7 cycles on the 65816, 8-14? cycles on the m68000, and only 2 on the ARM7TDMI, the other processors are doing math on the addresses that needs to be done explicitly as a separate instruction on the ARM. So something that takes 1 instruction on the m68000 may take 1-6 ARM instructions, for instance.

Of course, if you organize your data properly, this has a smaller impact.

Still, even with all of these limitations, the ARM7TDMI could handle what the others simply couldn't: it could render basic 3D kind-of in real-time. It could run DOOM. And Duke Nukem 3D. And Tomb Raider. And, and, and.

In the case of the Sonic port, my thought is they probably treated the ARM7TDMI in the GBA like an M68000 in the Genesis. To get comparable speed they'd likely have to have done extensive re-structuring to the code. Also, the GBA has no APU, so getting the iconic music output also ate into the processing time.

### Further notes on each system...
Aside from raw speed, the architecture of the system has a lot to do with how its processor is used.

Here are a few notes on some common tasks...

#### Writing to VRAM
* SNES : 8 bits at a time, at full 3.5MHz speed. Only in hblank and vblank. This happens through a port, which means you must first set the destination address separately.
* Genesis: 8 bits are transferred at a time at a fairly fast speed, allowing CPU to blast 16-bit writes if it wants. Full-speed in vblank, reduced speed during screen and hblank. Like the SNES, this happens through a port, which means you must first set the destination address separately. 
* GBA: Transfers are 16-bit and can happen anytime. If the PPU is accessing the RAM at the same time, it adds a 1-cycle delay. VRAM is mapped in the CPU address space, which means it is accessed directly without the overhead of a port. Finally, any 8-bit writes go through as a 16-bit write repeating twice. Several 3d games used this to easily double pixels horizontally.

#### Handling sound
* SNES: had a CPU and hardware to offload this to (the SPC700 and the SNES APU)
* Genesis: had a CPU and hardware to offload this to (the Z80 and the SN7649 and YM2612)
* GBA: had basic 3-channel audio hardware. Requires CPU intervention to even play, let alone mix more than that. Also had 2 squarewave and a noise channel like the GameBoy.

#### Bitmapped output (like for 3D or arbitrary 2D)
* SNES: didn't have a bitmapped mode. Required drawing onto tiles in a semi-complicated way. Indirect access to VRAM further hampered things
* Genesis: didn't have a bitmapped mode. Required drawing onto tiles in a semi-complicated way. Indirect access to VRAM further hampered things
* GBA: had a few bitmapped modes, that could even be rotated and zoomed. Direct access to VRAM was helpful. Also MANY less pixels to draw overall

#### Raster effects
* SNES: DMA hardware interrupts the CPU to change display parameters. Per-line interrupts from the CPU are also possible.
* Genesis: Display parameters are mostly pre-calculated, though per-line interrupts from the CPU are also possible.
* GBA: More like the SNES than the Genesis

### Overall
Overall, the ranking is...

1. ARM7TDMI (GBA) - 2 points
2. M68000 (Genesis) - 1 point
3. 65816 (SNES) - 0 points

I'd say that, in terms of raw power, the GBA CPU is in practice only 2-4x more powerful than the Genesis, and with bad enough programming, could actually run slower. The Genesis is only about 1.5-2x as powerful as the SNES. This is a VERY GENERAL OVERVIEW, however, and with careful programming, the SNES can approach the Genesis in processing. It also doesn't take into account the GBA's lack of sound hardware.