## How to get to GameGear from SMS

### A technical post
So you've got an SMS emulator, like a bunch of people I know, and are interested in making it a GameGear emulator!

Unfortunately, the info about what exactly the differences are is kind of thin on the ground.

This blog post outlines exactly what changes you need to make to start running GameGear software today. It's super easy.

### Full GameGear?


### Basics and display
So, the basics.

First off, the GameGear is a very slightly upgraded Master System. Although its resolution is 160x144, it draws the entire NTSC screen. Sprites count even if they're outside the viewable area. It just takes the middle 160x144 pixels of the screen and displays them.

The GameGear has two modes, GameGear mode and SMS compatability mode, which is triggered with a physical cartridge feature. In SMS compatability mode, it does scale the whole screen down to 160x144, but I am not familiar with exactly what this entails.

The upgraded areas are:

1. A start button
2. It has 12-bit CRAM instead of the SMS's 8-bit, allowing for up to 4,096 shades for colors!
3. The sound chip has an additonal register to control sound output to each speaker independently, though it does not affect the built-in speaker (only the headphones)
4. There's a bunch of link cable-related ports added, which I mostly ignored

And some changes:
1. BIOS is not in first model GameGears, so is not mandatory
2. No Sega Card can be mapped, and AFAICT the GameGear cart cannot be unmapped. You should ignore the 
3. GG carts use the same Sega mapper as SMS carts

And a change in presentation...
1. The VDP now has 2 modes, SMS and GG.
2. SMS mode is set by a physical switch due to the cartridge shape
3. SMS mode works exactly the same, but the output is shrunk? Not sure
4. GG mode works EXACTLY the same as SMS (with exception of CRAM, see CRAM below). You just need to crop your output to the middle of the image. All interrupts, timing etc. is exactly the same.

### Discriminating between them
The GG's support for SMS backwards compatability, and the fact it is a physical switch on the cartridge, lead to an important considerations.

1. You shouldn't need to worry about SMS compatability mode. 
2. There is no flag in the game header saying it should be SMS or GG. Therefore I recommend you use a cartridge database like [S8BL](https://github.com/raddad772/s8bl) to determine extra info about a cart.

### VDP behavior

#### CRAM
 
CRAM or Color RAM is where the palettes for the SMS/GG are stored. For an SMS emulator, it's fine to use 8 bits to represent this. For a GameGear emulator, I recommend you use at least 16.

If we're in GameGear mode, writing to CRAM is a little different.

```
If write to even address, set latch to value written.
If write to odd address, set CRAM[address >> 1] = ((value & 0x0F) << 8) | latch;
```

Meaning that writes to even CRAM addresses set a latch value, and odd addresses uses it to write 12 total bits to an actual color.

If bit 3 of VDP mode control register is clear, we read from the pre-SMS compatability palette. Otherwise, straight from CRAM.


#### Display output

Other than this, VDP behavior is basically identical and needs no changes (although remember output to the LCD is scaled down for SMS compatability mode, or cropped for GG mode).

The scaling needs further investigation. The cropping is slightly complicated by the fact the VDP can be in 192 or 224-line modes. To get the correct top and bottom lines to DISPLAY (since we render the full SMS frame), do this:

```javascript
top_line = (bottom_line - 144) >>> 1;
bottom_line = top_line + 144;
```

And you need to remember to get the colors of the ouput buffer in the new GameGear format. Since it's 4 bits per component, this is super duper easy to do. Just multiply each component by 0x11!

```javascript
b = ((color >>> 8) & 0x0F) * 0x11;
g = ((color >>> 4) & 0x0F) * 0x11;
r = (color & 0x0F) * 0x11;
```

#### Screen difference
The GameGear has a 160x144 screen with a 4:3 aspect ratio, meaning that the pixels are NOT square. Art had to be readjusted to go from SMS (different not-square aspect ratio) to GG without being distorted.

If you don't account for the 4:3 aspect ratio, circular elements will look oval, etc. 

### I/O ports
#### Link Cable
The GameGear has additional I/O ports dealing with the link cable that I did not bother with; info for these can be readily found. On read, ports 1-7 return 0 for me.

#### System Control 0x3E
The GameGear should ignore the cart enable bit and always act as if it's true. Games will try to disable the cart; I'm not sure why, but you must ignore this.
I also ignore the BIOS enable bit because I don't use the BIOS (yet), but I have no idea if this is correct.

#### Controllers & Input
The GameGear has no Reset button.

It does have a Start button for p1. Port #0x00 should return 0x40 OR the start button status in bit 7.

```javascript
return 0x40 | (gg_joymap.start ? 0x80 : 0);
```

Every other port that I can find acts identically, with P2 controller reading as disconnected.

No need to modify any other port behavior except 0x3E.

### That's it?
I think so. I went over the source to my emulator and that's all the differences I can find. However, I did not emulate the stereo outputs for sound, which can be found here and are super simple. [Link to document](https://www.smspower.org/Development/GameGearStereo)

If I left anything out or something doesn't work for you, please open an [Issue](https://github.com/raddad772/raddad772.github.io/issues) on this repository! Thanks and good luck with GameGear!
