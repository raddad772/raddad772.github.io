## SNES myths

---

### Intro
I'm going to be collecting some SNES myths and misunderstandings here, and the truth of them. I will update this over time.

### Nintendo chose a really poor processor
The WDC65c816 was actually shaping up to be a pretty good processor. Even if it didn't have some of the built-in features of the M68000 like integer divide and multiply, it was still based on a fairly decent architecture (6502) and could be extended (as it was - Ricoh added a multiply/divide unit to it). It was designed to run at a competitive speed, around 8-10MHz.

However, due to a bug in the core, running the 65c816 faster than about 3.5MHz would cause serious bugs. Basically, some traces inside were too long, and signals from two instructions (REP & SEP) would take too long to propagate. WDC later fixed this and, in fact, the SA-1 "accelerator" chip found in many carts is basically a 65c816 with this fixed, clocked around 10.7MHz. It makes a huge difference to games that use it.

### The SNES CPU runs at 3.58Mhz/2.68MHz/1.79MHz
The CPU of the SNES is a Ricoh 5a22. This is a custom-manufactured processor, which includes a WDC 65c816 CPU core, a DMA/HDMA controller, interrupt timers, address decoding logic, a DRAM refresh circuit, a multiply & divide unit, and more. This processor runs at 21.477MHz.

Each of those cycles is known as a "master cycle." A clock signal is composed of two phases, a high and a low phase. The 5A22 dynamically changes the length of a 65c816 cycle in this way.
Phase 1 is *always* 3 master clock cycles.
If the CPU is doing an internal operation, or accessing FastROM, or Bus B (where the PPU and other components live), Phase 2 is 3 cycles.
If the CPU is accessing SlowROM or WorkRAM, Phase 2 is 5 cycles, to give time for the RAM or ROM to respond.
If the CPU is accessing certain slow parts internal to the 5A22, Phase 2 is 9 cycles.

In that way, for every 6, 8, or 12 master clock cycles, we get 1 65c816 cycle. This is the source of the 3.58/2.68/1.79MHz numbers. These numbers are very misleading, however, because the 5a22 does alot of other things that pause the CPU in order to accelerate processing, such as DMA and HDMA transfers.

### ROMS or RAM can be accessed faster than their rated speeds through PPU WorkRAM registers
Although it's true that the PPU WorkRAM registers do, in fact, allow 6-master-clock-cycle access to slow ROM which normally takes 8 master clock cycles, that is referring to the length of the one CPU cycle that does the actual load. The CPU cannot read from one address faster than 8 master cycles - by the time it is time for a new read from the WRAM port, more than 8 master cycles have elasped. If you need to repeatedly load A by scanning through memory, you can shave off a few cycles this way, but you're not actually accessing RAM or ROM faster than the chips allow. The fastest read instruction that can reach the appropriate address, LDA absolute, takes 4 CPU cycles, or a minimum 24 master clock cycles. Even the DMA controller only changes the address lines every 8 master clock cycles, respecting the speed of slowROM and the RAM.
