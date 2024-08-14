## Genesis VDP internals, part one

So I've just more or less finished writing a very accurate M68000 core. I already have a highly-accurate Z80 core. What better to do than work on a Genesis emulator?

*What's that, Dreamcast emulator? I could work on you? Shush, now!*

Unforunately, if you want to make an accurate VDP core, you pretty much have to put together information from a bunch of different random forum posts. Many of which are outdated and superseded. If only there was a blog post that would go over it in depth...

__THIS IS A WIP. Until I remove this line, don't trust this!__

### VDP?

So what is a VDP, you're asking? It's the part of the Genesis that generates the display. It stands for Video Display Processor, and it makes all the pretty colors.

I'll be using Ares source to do a deep dive, because it's fairly well-regarded and up-to-date with current Genesis understanding.

### The basics

A few things to get to first.

VDP has 64K of RAM. The interface to the VDP is 16-bits, which helps with bandwidth.

The Sega Genesis master clock on NTSC is 53.693175 MHz, and 53.203424 on PAL. The main CPU runs at a clock divisor of 7, the Z80 runs at 15, and the VDP at 4. So every 4 "master cycles," a cycle is performed on the VDP, and every 8, a pixel is generated if we're in the right place.

The VDP conforms to the NTSC standard. I'll be mostly discussing NTSC here, though I may note PAL differences.

If you're not familiar with analog TV frames, VERY basically, on each line there is a visible and a non-visible portion. The non-visible portion is known as the "horizontal blank," and includes the time the elctron gun takes to trace from the right of line a to the left of line b.

Similarly, each frame has visible areas before and after the visible lines. When we're not displaying, we're in vblank or vertical blank. A portion of this time is used to move the electron gun from the bottom of the screen to the top.

The Genesis VDP doesn't get to pause in the middle of rendering. It needs to output its pixel data on time, every time. It needs to efficiently use the bandwidth it has to generate a picture.

This is similar to the way the NES, Master System/Game Gear, and other consoles work, but the VDP is more advanced than those.

### The slightly less basics

The VDP can do a lot of cool things. It has 2 conventional tile planes that can be scrolled, a background plane, and a sprite plane. Each of these can display up to 15 out of 512 colors (notwithstanding shadow and highlight), and if nothing is displayed, a default background color is output.

Tiles can be scrolled in flexible and interesting ways. Data can be moved into VRAM via DMA, which is significantly faster than having a processor do the work.

### Pixels!

To output a pixel, the VDP has to choose between (and using glitches, blend) the different layers. The background color, layer A, layer B, and sprites are all evaluated by priority, depending on mode. 

To do certain transparency effects, some games would use the debug register. The debug register is a special register that could force non-standard behavior. For instance, it could force a layer to be visible that otherwise may be hidden or disabled. When more than one layer was forced this way, their color values would be logical ANDed together, although this is relying on analog behavior that didn't work well in the earliest VDP revisions.

### A more technical dive into what happens each pixel...
For each pixel, layer A, B, and sprites are rendered.

The aforementioned "window" layer is basically a low-functionality extension to Layer A. At certain times during drawing Layer A, if something is transparent, it can take the pixel from the background instead.  