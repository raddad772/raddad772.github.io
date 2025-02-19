## Notes on GBA PPU: Windows and Blending

EDIT 2/19/25: fix some small accuracy issues in terminology

I've been working on GBA lately. I struggled a LOT with this. I found gbatek and even TONC's explanations to be confusing vs. the relative simplicity of what's actually going on.

### Rendering a GBA picture
A GBA frame consists of the following elements: 

1. Sprites (rendered to a line buffer one line in advance by the hardware. I just do it at the start of rendering that line, since I go line-by-line). They can be individually marked as transparent, and they can be regular or affine (rotatde/scaled), but affine ones take a bit more than twice as long per-pixel to draw, so you get less per line.
2. Backgrounds 0-3. They can be "regular" tile-mapped, 4bpp or 8bpp, or 8bpp affine. You lose 2 "regular" backgrounds for each "affine" background layer. Also there are bitmap modes, which basically just replace bg0-3 and follow most rules
3. Windows. These are conceptual things that can do things like enable or disable color effects, and keep certain elements from being drawn.

There are also 3 color effects:
1. Alpha blending, where two targets are blended together
2. Brightening, where one target is brightened
3. Darkening, where one target is darkened

Sprites and backgrounds can have priorities from 0-3, with 0 being the highest.

### Drawing a pixel
To draw any given pixel, the GBA has to...
1. Have rendered the sprites for that pixel previously,
2. Render backgrounds 0-3 if enabled,
3. Combine this all together with windows and sampling to decide what the output is.

That last step especially I got very confused about.

### More on windows
There are 4 windows:
1. WIN0, just a regular window, where the area "inside" is defined by top, left, right, and bottom
2. WIN1, same as WIN0
3. WINOBJ, where what is inside is defined as where any sprite pixels that aren't transparent, drawn in mode 2, are.
4. WINOUTSIDE, which is basically "everything not contained in another window."

WIN0, WIn1 and WINOBJ can be enabled or disabled separately. WINOUTSIDE is only enabled if any other window is enabled, else no window is enabled.

For each pixel we're rendering, each ENABLED window can have a status of "inside" or "outside." Consider a flashlight illuminating a wall: the circular area the light falls on is "inside" the window.

The window also has some bits associated with it:

1. OBJ - controls if sprites are made visible by this window
2. BG0-3 - controls if those backgrounds are made visible by this window
3. color FX - controls if color effects are enabled by this window

### Priorities
Each background can have its priority set individually, as can sprites, from 0-3.

A priority 1 sprite will always display in front of a priority 2 anything else. A priority background will display in front of priority 1-3 anything else.

If two elements have the SAME priority, then this order is used: sprite, bg0, bg1, bg2, bg3

### To actually draw a pixel
#### Elements
Once we've rendered the sprites and backgrounds 0-3, we are left with 6 pixels to decide between, in order of priority (assuming they were all set to the same priority):

1. Sprite pixel
2. BG0 pixel
3. BG1 pixel
4. BG2 pixel
5. BG3 pixel
6. The background color, defined as palette RAM entry 0. (Note you cannot adjust this priority, and ANYTHING that renders a pixel is always ahead of this)

We have that info. We have the states of the windows, and the color blending configuration. Now...how do we determine what value to output!?

#### First, determine "active window"
Only one window applies at a time (or no window).

The "no window" will act like a window with all backgrounds and objects enabled, as well as color FX enabled, where every pixel is inside it.

We want specific info from the active window for each pixel: if OBJ, BG0-3, and special FX are enabled.

First, we set all of those enables to true

```python
actives = [obj: True, BG0: True, BG1: True, BG2: True, BG3: True, SFX: True]
```

Then we check if any windows are enabled...

```python
if window0.enabled || window1.enabled || window_obj.enabled:
```

Then, we check in order of priority if they apply to this pixel:
```python
    actives = window_other.actives   # Default to this
    if (window_obj.enabled and window_obj.this_pixel_is_inside) actives = window_obj.actives
    if (window1.enabled and window1.this_pixel_is_inside) actives = window1.actives
    if (window0.enabled and window0.this_pixel_is_inside) actives = window0.actives
```

Once we have done this, we now know which features are enabled for this pixel in terms of OBJ, BG0-3, and Sprite FX.

#### Blending modes
The blending mode can be set in the PPU registers to 4 values:

0. Normal - no blending - output highest-priority pixel
1. Alpha
2. Brighten
3. Darken

Conceptually, there are then 2 "blend targets" possible. I will refer to these as blend target A and B, or Above and Below if you want to.

OBJ, BG0-3, and color0 can all be enabled or disabled individually for each blend target. So Blend Target A could say bg1+obj, and Blend Target B could be set for b2+bg3.

Before we get into choosing a target, it's important to understand a few things, it simplifies things a LOT:

1. Normal, brighten, and darken only require 1 target
2. For Brighten and Darken, the candidate pixel will be the highest priority pixel. It must be a valid Blend Target A, if it is not, brighten/darken will not happen, pixel will output as if mode 0.
3. For Alpha, the first pixel must be a valid Blend Target A, and the second must be a valid Blend Target B. Furthermore, they must be the highest 2 priority pixels (except in the case of transparent sprites, we'll get there, and this is a simple special case). If any of this is not true, blending will fail and it will output as if mode=0
4. SFX must be enabled by the window, if there is a window, for any mode other than 0
5. There is a caveat with Alpha. If we have a transparent sprite pixel as target A, it will work even if color SFX are disabled, assuming other conditions are true  

So an algorithm to determine which pixel to output would look like this:

```
#determine the two highest-priority pixels that are enabled by the current window or lack of window into target_A and target_B

output_color = target_A.output_color

if (special effects is enabled by the window or not having a window), OR\
   (the highest-priority pixel (target_A) is a translucent sprite pixel, and the second-highest-priority is a valid Layer B target):
    if (target_A is a translucent sprite pixel, and the target_B is a valid layer B target):
        output_color = blend(target_A, target_B)
    else if (mode == alpha_blend, and target_A is a valid layer A target, and target_B is a valid layer B target):
        output_color = blend(target_A, target_B)
    else if (mode == brighten, and target_A is a valid layer A target)
        output_color = brighten(target_A)
    else if (mode == darken, and target_B is a valid layer B target)
        output_color = darken(target_A)  
```

And determining the two highest-priority pixels is pretty easy too! Here's a simple algorithm for it:
```c
// Where... 0 = OBJ, 1-4 = BG0-3, 5 = color0 (backdrop). These are ordered by priority first
int target_a = 5, target_b = 5 // Set our targets to their defaults, which is color0 (backdrop) 
for (int priority = 3; priority >= 0; priority--) { // search priorities lowest to highest (0=highest) 3, 2, 1, 0
    for (int layer = 4; layer >= 0; i--) { // search layers lowest to highest (0=highest) 4, 3, 2, 1, 0
        if (layer_has_pixel(layer) && layer_priority(layer) == priority)) {
            target_b = target_a;
            target_a = layer;
        }
    }
}

```

NOTE: a layer has no pixel if the color is transparent, or the layer is disabled, or the layer is not enabled by the window.

It may seem a lot to do this for every pixel, but it is correct and runs fine in my emulator at least.

## Whew!

Following this process is the simplest way I could come up with to get correct output. I came to it after 3 iterations of much more complex logic that kept getting it wrong. So far, it seems to match what a GBA does!

For more info on configuring windows, blending modes, what they do, etc., of course check out GBAtek!
