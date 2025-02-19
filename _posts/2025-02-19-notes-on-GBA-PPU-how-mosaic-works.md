## Notes on GBA PPU: How mosaic

### Rendering a GBA picture
One of the visual FX available to the GBA PPU is a "mosaic" mode. But how do you implement this?

### First off, a note
Horizontal mosaic is actually applied as a post-processing effect during output of the line, just before the blending stage. It's actually fairly simple how it's done, too!

But first let's address vertical mosaic, as it will illuminate a few points for us.

### Vertical mosaic how-to
If we think of screen rendering happening like this:

```c
bg0.render_line(screen_y);
bg1.render_line(screen_y);
bg2.render_line(screen_y);
bg3.render_line(screen_y);
sprite.render_line(screen_y);
```

Vertical mosaic just changes it to this:  

```c
bg0.render_line(bg0.mosaic_enabled ? bg_mosaic_v_counter : screen_y);
bg1.render_line(screen_y ? bg_mosaic_v_counter : screen_y);
bg2.render_line(screen_y ? bg_mosaic_v_counter : screen_y);
bg3.render_line(screen_y ? bg_mosaic_v_counter : screen_y);
sprite.render_line(screen_y ? obj_mosaic_v_counter : screen_y);
```

The line is rendered just as if it were the line held in the _mosaic_v_counter variable (with a caveat for affine backgrounds that we'll get to).

### H and V counters...how they work
For background and sprites, the GBA keeps separate H and V "mosaic counters." These are incremented by N every N lines or columns. These counters are then used as a parameter in rendering.

So for a mosaic value of 3, the value for each row of the screen will be like this:

```
Row 0: 0
Row 1: 0
Row 2: 0
Row 3: 3 - n(3) rows have gone by, so increment by n(3)
Row 4: 3
Row 5: 3
Row 6: 6
Row 7: 6
Row 8: 6
Row 9: 9
and so on
```

The counter works exactly the same on the horizontal.

A simple, very easy way to accomplish this is like so:

```python
if mosaic.bg.y_counter == 0:
    mosaic.bg.y_current += mosaic.bg.y_reload   # the mosaic value such as 3
    mosaic.bg.y_counter = mosaic.bg.y_reload
mosaic.bg.y_current--
```

## Horizontal mosaic
When I first started implementing horizontal mosaic, oh my heavens was my implementation method...contrived.

It turns out the GBA just renders each line normally and applies mosaic as a sort of post-processing effect.

On the hardware, there's a sort of a "latched pixel" for each background, that holds the value of the pixel to compare against windows, do blending on, etc.

For backgrounds, it's very simple. This "latched pixel" is only updated with the just-rendered pixel when mosaic.bg_h_counter == 0 (so every pixel for mosaic=1, every other for mosaic=2, etc.)

That couldn't be simpler, right?

Sprites are a TINY bit more complicated, though.

### Sprite horizontal mosaic
But not a lot more complicated!

The additional complication comes in because of transparent sprite pixels, and also, sprites being per-sprite mosaic.

Our line buffer for rendering sprites (on real hardware this is rendered the line before the currently output one), contains info for each pixel:

1) If a pixel was rendered here at all,
2) What color the pixel is,
3) What priority the pixel is,
4) If the pixel is from a translucent sprite

All this is needed for a good output. However, to get good sprite mosaic output, we also need the following info:

5) If the pixel is considered mosaic

Note my phrasing there. "If the pixel is considered mosaic." What?

On GBA hardware, sprites are rendered from 0->127, in order, onto the buffer. If a pixel already exists and we try to output a new one, we do not unless we are a lower priority, and are opaque.

HOWEVER. In order to properly blend (AND to properly do mosaic), if we are a lower priority, we ALWAYS overwrite the mosaic and priority attributes for a pixel, even if we are not an opaque pixel.

My code for this is a little fugly and looks like this:

```c
if ((mode > 1) || (!outpx->has) || (priority < opx->priority)) { // If we are mode 2, or the output pixel is empty, or we are a higher priority...
    switch (mode) {
        case 1: // Translucent
        case 0: { // Normal
            if (c != 0) { // If we are opaque, set these attributes
                outpx->has = 1;
                outpx->color = this->ppu.palette_RAM[c + palette];
                outpx->translucent_sprite = mode;
            }
            outpx->priority = priority; // ALWAYS set these attributes
            outpx->mosaic_sprite = mosaic;
            break;
        }
        case 2: { // window sprite
            if (c != 0) w->inside[sx] = 1;
            break;
        }
    }
}

```

Now this is important because it preserves the ability of a sprite to produce a mosaic look even if there's a lower-priority sprite in that particular pixel. Otherwise, we couldn't choose properly.

### Now the sprite pixel output

When choosing whether to update a background's "pixel latch," we simply looked at if mosaic_counter == 0. The sprite is similar but a bit more complex.

For sprites, the rule is that we update the pixel latch on a given pixel IF:
1) The currently-latched pixel does not have mosaic marked
2) The pixel we may update with, does not have mosaic marked
3) mosaic_counter == 0

```c
if (!latched->mosaic_sprite || !src->mosaic_sprite || (mosaic_counter == 0))  {
    update_latched_pixel here
}
```

This allows a higher-priority mosaic sprite to properly cover up a lower-priority sprite.

Congrats on your awesome-looking mosaic effect!