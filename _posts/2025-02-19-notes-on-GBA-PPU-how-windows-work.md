## Notes on GBA PPU: How windows work

### Rendering a GBA picture
As we've discussed previously, GBA has these 4 objects called "windows." Imagine it like a flashlight illuminating a wall in a dark room: the area inside the light is "inside" the window.

These 4 windows "exist":

1) WIN0, the first window, enabled/disabled separately
2) WIN1, the second window, enabled/disabled separately
3) WINOBJ, a window defined by opaque sprite pixels drawn in mode 2, enabled and disabled separately.
4) WINOUTSIDE, literally just anything not "inside" any other window, and ALWAYS enabled if any other window is enabled

In a previous post, we discussed how to determine which window applies to any given output pixel. To determine this, we needed to know if a pixel counted as "inside" for a given window. But how do we determine that?

### The way hardware does it
Hardware does this in an EXTREMELY simple manner.

In hardware, WIN0 and WIN1's position is defined by its top, left, right, and bottom coords.

They also have a flag that works like this.

```python
def for_each_pixel(window):
    if screen_y == window.top: window.v_flag = True
    if screen_y == window.bottom: window.v_flag = False
    
    if screen_x == window.left: window.h_flag = True
    if screen_x == window.right: window.h_flag = False
    
    window.pixel_is_inside = window.enabled and window.h_flag and window.v_flag
```

Note this is also done during vblank!

This extremely simple scheme is actually fairly flexible and allows for some cool effects.

### What WINOBJ and WINOUTSIDE?
Determining the flag for WINOBJ depends on the particulars of your rendering setup; it has to do with if an opaque pixel was rendered for a sprite in mode 2 on that particular pixel or not.

For WINOUTSIDE, the expression is also fairly simple:

```python
WINOUTSIDE.pixel_is_inside = (not WIN0.pixel_is_inside) and (not WIN1.pixel_is_inside) and (not WINOBJ.pixel_is_inside)
```

