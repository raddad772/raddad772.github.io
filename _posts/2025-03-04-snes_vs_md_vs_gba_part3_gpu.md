## Super Nintendo vs. Genesis/MegaDrive vs. GameBoy Advance, part 3: GPU

I've decided to order these in order of power from least to greatest. This one is much more clear-cut than the CPU section was. 

Remember, the most important metric is really how much data can be processed how fast. 

### The specs
On consoles, screens are made up of "sprites" (movable objects that can be positioned anywhere), and "backgrounds" which are made up of tilemaps (or on the GBA they can also be bitmaps).
They are composed out of a certain palette of colors, and can have various special effects used on them.

* Genesis:
  * 64 KB 16-bit VRAM
  * Pixels
    * 256 or 320-pixel horizontal resolution, 224-pixel vertical. That's 57,344 or 71,680 pixels per frame
  * Colors
    * Up to 62 (1 background, then 15 from each of 4 palettes) colors on-screen without tricks
    * From a 9-bit (512-color) palette
  * Display modes
    * DISREGARDING old Master System modes, only 1 display mode. 2 independent backgrounds @4bpp, plus a sprite layer 
  * Sprites:
    * 4 bits per pixel"
    * 64 or 80 sprites processed per line depending on horizontal pixel resolution
    * Up to 20 can be visible on a line
    * Up to 32 or 40 sprite tiles drawn max depending on horizontal pixel resolution, or 256 or 320 pixels of sprites
    * Can be any arbitrary combination of sizes up to 4x4 tiles (64x64 pixels)
  * Backgrounds:
    * 4 bits per pixel
    * 2 independently scrolling layers
    * Scrolled in 16-pixel columns vertically. Per-column scrolling does not play nicely with horizontal scrolling
    * Scrolled in 1-pixel rows horizontally
    * 1 bit for priority led to relatively fixed ordering
  * Special effects
    * One background has a "window mode", which basically forces reading from different addresses on an 8-pixel horizontal boundary. This also leads to some scrolling bugs if horizontal scrolling is used. This is basically for score and status bars.
    * There is a shadow/highlight mode, where the RGB output values are halved to darken them, and then .5 is added to each one for lightening them. Certain combinations of background and priority can be "normal" or "dark," and certain sprite colors act as a "transparent" "lighten." This leads to very primitive darken/lighten effects that are used for some shadows, hi-color effects like the title screen of Toy Story, "scanner" effects, etc.
* SNES:
  * 64KB 16-bit VRAM
  * Pixels
    * 256 or (rarely used) 512-pixel horizontal resolution, 240-pixel vertical. That's 61,440 pixels per frame or 122,880 in higher-res mode
  * Colors
    * Up to 256 colors on screen at a time
    * From a 15-bit (32,768) color palette
  * Display modes
    * 7 common display modes, that led to various combinations of 1-4 backgrounds at 2bpp, 4bpp, or 8bpp
  * Sprites:
    * Up to 128 sprites processed per line
    * 32 max sprites visible on a line,
    * 34 slivers (tiles) max per line, or 272 pixels of sprites
    * Can be a few combinations of sizes
  * Backgrounds:
    * 4 backgrounds @ 2bpp
    * 3 backgrounds with one at 4bpp and tow at 2bpp
    * 2 backgrounds with one at 8bpp and the other at 4bpp, both at 4bpp, or one at 4bpp and the other at 2bpp
    * 1 background at 8bpp with less effects like tile flip. But affine transformations (skew, rotate, zoom). This is mode 7
    * Independent horizontal and vertical scrolling
    * Some modes have per-column vertical scrolling like the Genesis
  * Special effects:
    * Mosaic. This is the "blocky" effect.
    * Color math. Layers can be added, subtracted, averaged, and more to create many different color blending effects
    * Windows. Pixels can have their properties changed by a virtual "window" that can be any shape. This can produce a0 lot of different effects.
    * Mode 7 affine transformations (skew/rotate/zoom). This can be used skillfully for a pseudo-3d plane a la mario kart, f-zero, etc.
* GameBoy Advance:
  * 96KB 16-bit VRAM
  * Pixels
    * 240x160 or only 38,400 pixels per frame!
  * Colors
    * Up to 510 in "classic" modes, up to 32,768 colors in bitmap modes at a time
    * From a 15-bit (32,768) color palette
  * Display modes
    * Also 7 modes, but different from SNES. Combinations of 1-4 backgrounds, tiled or bitmapped, possibly with affine transformations, at 4 or 8bpp
  * Sprites:
    * 4 or 8bpp color
    * Up to 128 sprites processed per line
    * Sprites can be regular (like other consoles) or affine (transformed/rotated/skewed)
    * Sprites can independently be blended and work as "window layers"
    * Sprite limits on the GBA work a bit differently. Instead of a max number of sprites and tiles drawn, each sprite processed takes 1 cycle for regular or 10 for affine, then 1 cycle per pixel or 2 for affine. This is out of 1210 cycles if "hblank VRAM access" is disabled, or 954 if it is enabled.
    * So if only regular sprites, up to about 148/116 tiles can be drawn per line, or about 75/58 affine ones, for 1210/956 normal pixels or 605/476 affine pixels.
    * Can be many combinations of sizes
  * Backgrounds
    * Tiled backgrounds can be 4 or 8bpp
    * Bitmapped backgrounds can be 8 or 16 bpp, and can be single- or double-buffered (though the amount of VRAM puts a restriction on the size of double-buffered 16bpp to be 160x128)
    * Each "affine" (transform/skew/zoom) background takes up 2 "tiled" backgrounds
    * So, 4 tiled backgrounds, or 2 tiled and 1 affine background, or just a bitmapped background
    * Sprites are drawn on top of the bitmapped backgrounds, as well as blend operations, and they can be affine-transformed
  * Special effects
    * "brighten," "darken," and "blend" color blending available - best of both Genesis and SNES
    * Mosaic like SNES
    * Windows like SNES
    * Affine transformations like SNES mode 7, with bitmaps or twice as many backgrounds, and more detail 

OK. So we have a bunch of specs. It's obvious the GBA has the best graphics, as it can do everything the SNES and Genesis can and a lot more.

But between the Genesis and SNES, they each have strengths.

### Genesis vs. SNES
Although I've listed the SNES PPU as overall more powerful due to its ability to have more advanced graphical effects, even if you took out Mode 7 I'd say the same thing.

However, the Genesis does have some advantages over the SNES and even the GBA!

#### Screen crunch
The Genesis is capable of rendering 256- or 320-pixel lines. Although the SNES is technically capable of 512-pixel lines, this was VERY rarely used outside of some text rendering, and mostly only ever uses 256-pixel mode.

On a TV screen, the size of the line didn't change because you had more pixels: the amount of detail packed in did. Since they were both based on 8x8 tiles, that means that the Genesis could display more horizontal detail than the SNES. 

One important note there, however, is that although it had greater LUMINANCE (light/dark) detail, the "color resolution" stayed the same, and was in fact below 256 pixels. This is due to the color carrier of the TV signal. Modded consoles with RGB output or higher-quality outputs could lead to more detail. This was exploited by some games, such as Sonic the Hedgehog's waterfalls. Later consoles like the Saturn would exploit this property to blend the colors together and create faux transparency.

What all this means, is that the Genesis could show 25% larger, more detailed playfields in the same space.

The GBA was a little worse at 240 pixels, although it's worth noting that it had full luminance and color resolution.

#### Sprites
The SNES has more overall sprites (128 vs 80), but who used 128 sprites? Perhaps some SNES games did, since they weren't as flexible as the Genesis'.

Each sprite on the Genesis could be ANY combination of 1x1-4x4 tiles. This allowed for simpler management of complex shapes by games. Big, detailed sprites would require less of the hardware "sprite slots" than the equivalent on the SNES. This was easier on CPU time and programmers as well. This is why you see some pretty funky-looking distorted sprites in some cross-platform games. 

The SNES could actually draw a bit more sprite pixels per-line than the Genesis in 256-pixel mode: the Genesis did 256, and the SNES could more than cover a single line with 272.  But this jumped up to 320 in 320-pixel mode.

#### Everything else
In almost everything else: amount of colors available on screen, special effects available, color depth, number of backgrounds, etc. - the SNES beats the Genesis pretty handily, and GBA wins. This is why I gave the victory of SNES vs. Genesis to SNES.

For some people, the screen crunch is a deciding factor: the 25% more display area of the Genesis can make a big difference in some games. And I respect that. However, I still think the S-PPU is technically better than the VDP.

### Overall
Overall, the ranking is...

1. GBA - 2 points (2 already) = 4
2. SNES - 1 points (0 already) = 1
3. Genesis - 0 points (1 already) = 1

So GBA is in first, with SNES and Genesis tied behind it.
