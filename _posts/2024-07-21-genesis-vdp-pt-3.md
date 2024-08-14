## Genesis VDP internals, part three: basic timing, DMA, and VRAM access slots

Yes, part 3 coming before part 2. Part 2 should be a reference on a bunch of basic VDP stuff you can find just about anywhere, like the function of various registers. I honestly may never get to it. Go learn some basic VDP stuff and come back.

### A few terms
#### Clock divisor

What is a clock divider? It is a circuit that divides a clock signal. If you have a 100 Hz clock, a /4 divider will give you 25Hz. A /5 divisor will give you 20Hz. A smaller number is faster. /X divider means, every X clock ticks, this device will tick. The "clock divisor" is the amount it is divided by.

#### mclk
mclk stands for main clock or master clock. In the MegaDrive/Genesis, this is around 53MHz. Each component uses a clock divider to run at their own correct speeds.

#### VRAM

VRAM refers to the 64KB of VRAM the Genesis uses.

Interestingly, it was designed around 128KB of VRAM, but Sega cut that down to 64kb relatively late in development. The VDP still has a bunch of features that use it, and this choice roughly halves the DMA bandwidth available.

#### VSRAM

VSRAM is Vertical Scroll RAM, 20 9-bit entries internal to the VDP that hold vertical scroll values independently for 16 columns accross the screen.

#### CRAM
Color RAM is a 64-entry, 10-bit RAM that holds palette information.

#### Pixel Clock and Serial Clock (SC)
The Pixel Clock is the clock by why the VDP outputs pixels. This happens with a /8 or /10 divisor off the main clock. It is exactly double the SC.

The Serial Clock or SC runs twice for every pixel output. It runs at either a /4 or /5 divisor of the main clock. This is used to fuel VRAM reads and writes.

#### h32 and h40
H32 refers to 32-cell, 256-pixel-wide mode, while H40 refers to 40-cell, 320-pixel-wide mode.

### OK so back to the fun stuff

So, the VDP outputs a pixel every 8 or 10 main clock (mclk) ticks, depending on a few things.

Internally, the VDP has a /5 clock divisor. It is also capable of linking to EDCLK, an external clock at /4 divisor, to output pixels in 320-wide mode.

At all times, EXCEPT for during most of a scanline, the VDP is at the /5 divisor. It syncs to the /4 divisor for a portion of the scanline only. It does this all throughout the frame, even in vblank, when h40 is enabled.

The reason for this, is that a 256-pixel scanline takes the same amount of time to output as a 320-pixel scanline. So it has to output them faster in 320-pixel mode. But the hblank (the time the beam takes to reposition to the start of the next line) has to remain the same length of time.

### A bit more detail on scanline timing

In the most common video modes on an NTSC machine, any given scanline takes exactly 3420 master clocks.

Lots of things are happening each Serial Clock or SC tick, which remember, is twice as fast as pixels are output. In fact, any access to VRAM takes 4 SC ticks. In actuality it is 7, but the first and last 3 overlap, meaning that effectively it is 4.

Every 4 SC ticks, there is a "VRAM access slot." This is a place for the VDP to read from VRAM in some predetermined way, or for the VDP writes from DMA and the CPUs to be committed to RAM. The kind of slot for each 4 SC ticks is controlled by a PLA that decodes the specifics. I'm linking some images here that go into detail for an entire scanline. These images are produced by someone much more dedicated and skilled with Sega emulation than I am, and as always, we modern emulator authors are profoundly grateful for those who came before us and did the hard work of figuring all this out.

![Timings for H32](http://nemesis.hacking-cult.org/MegaDrive/Documentation/VDP/VDP%20VRAM%20timing%20H32%20-%20small.png)

![Timings for H40](http://nemesis.hacking-cult.org/MegaDrive/Documentation/VDP/VDP%20VRAM%20timing%20H40%20-%20small.png)

And the source forum post:

[VDP VRAM access timing](https://gendev.spritesmind.net/forum/viewtopic.php?f=22&t=851)

### Writing to VRAM, VSRAM, and CRAM
If you'd like to write to VRAM, VSRAM, or CRAM, you need to go through the VDP. VRAM is hooked up to the VDP, not either of the CPUs, and VSRAM and CRAM are internal to the VDP itself.

To write to these, you set up the address you want to write to as well as the auto-increment in VDP registers, and then send data to the VDP data port.

Although reads happen immediately and at any time, writes have to go through a 4-entry FIFO buffer. This design allows writes to VRAM even during active rendering, something not really allowed on Nintendo consoles of the era, although they're much slower than during vblank or when rendering is disabled.

When the FIFO buffer is full and something writes to it, DTACK is not asserted until the FIFO buffer gets to write at least one of its entries out. This causes the M68k and DMA unit to pause indefinitely until at least one "external access" VRAM slot happens, which can be quite a while if you start at the wrong time (see charts above).

During these external access slots, the command FIFO is emptied one at a time. However, it's not QUITE that simple. The VDP was designed around 128k of RAM, but only includes 64k. As a consequence, writes to VRAM have to happen 1 byte at a time, not 2 bytes at a time as you'd expect! If you write a word to the data port, it will take 2 external access slots to complete. Bytes are written in low, high order, for some reason.

While rendering is disabled, this doesn't take very long. In fact, it can keep up with as much data as the M68K can push. However, during rendering, speed is reduced to a fraction of that, and the FIFO is easy to fill.

VSRAM and CRAM receive the whole word in one access slot.

### DMA

DMA is actually a pretty simple mechanism. It has 3 "modes":

1. 68k bus -> VRAM, VSRAM, or CRAM
2. VRAM -> VRAM
3. fill data -> VRAM

You basically set the destination up as if you were about to do a CPU write, and then trigger the DMA unit. It will then repeatedly read from whichever source (including a byte you write to it for fill mode) and write it to the data port. This will fill the FIFO buffer with write commands. It then pauses until the FIFO buffer discharges at least once and goes again.

I've had some trouble finding accurate information on exactly what commands cause DMA and which bits do what. Different reputable sources say to do different things. So, using the source of Ares, largely considered a fairly accurate VDP emulation, I've compiled the following information.


Data Port Writes


Control Port Writes