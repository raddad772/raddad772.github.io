## Super Nintendo vs. Genesis/MegaDrive vs. GameBoy Advance, part 1: Intro

I see a lot of questions about which was more powerful: SNES or Genesis. These often come from a good place, a place of curiosity about the underlying systems.

The answer, of course, is "it's complicated." But in diving in deep, we can uncover a lot of info about the systems.

I've emulated the SNES and the Genesis in some detail, so I set out to write a series of blog posts covering this topic in-depth. However, I think an even more interesting question is, how does the GBA compare to these?

Believe it or not, it's not a straight win for the GBA on all counts (I'm looking at you, sound), and some places are closer than you'd at first believe (cough cough, CPU).

So I'm going to be doing a deep dive comparing various subsystems of the GBA, Genesis/MD, and SNES.

### How!

If there's one thing that I've found emulating all these systems, it's that the most important thing isn't CPU speed, or RAM size, or number of transistors, or feature count. It's memory bandwidth. The actual amount of bytes that can be read to produce a picture on the screen, or the amount of data the CPU can move around and calculate. A superior ability to do this generally (though not always) results in a better picture.

An easy example of this is on the MD, where color RAM is only 64 entries of 9 bits. Color RAM is special RAM where color information is stored. A GPU (such as the S-PPU, GBA PPU, or Genesis VDP) has its hands full rendering the colors, and when it comes time to output to the screen, it must have the ability to transfer its limited palette data into an output color very rapidly. This is special RAM on its own bus which allows this. The Genesis only had 64 9-bit entries, leading to (without any tricks) only 62 colors on screen, from a 9-bit color space. This info takes up about 576 bits. The SNES, on the other hand, has 3,840 bits dedicated to 12-bit color, allowing a much easier profusion of colors on the screen.

Having less memory for this specific task really limits Genesis games in comparison to SNES ones. The Genesis is known for its relatively drab palettes and lack of colors on-screen in many games, and this is the root of why.

I'll try to look at each subsystem through these lenses, although it isn't always so simple. I will also try to include comparisons with the previous generation of the hardware - SMS for the Genesis, NES for the SNES, and GameBoy Color/GBC for the GBA.

I'll then feature a summary section that goes over key points and ranks each system.

As for a final score, I will reluctantly assign one. 1st place gets 2 points, 2nd gets 1 point, and 3rd gets 0 points. But that's fairly meaningless, really, and the GBA will almost always win anyway.

### Nomenclature
I will use 'GPU' to refer generically to the S-PPU of the SNES, PPU of the GBA, or VDP of the Genesis. The video-drawing hardware.

I will be use 'APU' to refer generically to the sound processing hardware, or APU CPU to refer to the dedicated audio CPU (if it is in the system - the GBA does not have one).

The systems have a lot of similar but different terminology like these. I'll try to use proper generic terms where possible.

### My credentials
I wrote my SNES emulator over 3 years ago now. I never got it past playing Mario World, Link to the Past and F-Zero. It was my first emulator since a bad NES one I made over 20 years ago.

Since then, I've emulated a lot of systems to various degrees. I bounce around a lot as I get distracted or motivated. Here's a list as of now:

* Apple II/e - doesn't boot, but has some hardware
* Atari 2600 - plays some games badly. Needs better video hardware support
* Sega Dreamcast - Boots the BIOS and plays homebrew. Does not have TA (3d drawing) implemented
* Galaksija - A Yugoslavian minicomputer. Fairly obscure. Seems to run fine, and very accurate, no tape support yet though.
* GameBoy/Color - Pretty accurate, runs most games, has decent sound. There are a few known incompatibilities, but I mostly lost interest
* GameBoy Advance - Not the MOST accurate emulator out there, but more accurate than most bother. Runs every game I've tried well with full sound etc.
* Sega Genesis - A VERY accurate emulator, missing a few things: YM2612 (FM sound) and proper VDP FIFO support (though VDP timing is, otherwise, correct AFAICT). 97+% compatability
* Mac Classic - Boots to sad mac, got disinterested
* Nintendo DS - Boots at least one commercial game. Very WIP. Mostly needs a lot of the 3D registers and hardware implemented.
* NES - Very accurate, high compatibility, though not perfect. Perfection for NES is a very long tail and I got disinterested in it.
* PlayStation - very WIP. My original one booted homebrew and had an OK GPU, but I stopped at the CDROM drive. The port I'm working on now is almost back to that point.
* Sega Master System/GameGear - fairly accurate, highly compatible
* SNES - Had lots of bugs, quite incomplete, and abandoned 3 years ago. If I get interested in SNES again I will likely just start over with my improved knowledge
* ZX Spectrum - a fairly accurate and good emulator

So I know a thing or two about emulation. I've also authored several sets of standard tests used by new emulator authors nowadays, for the following processors:

* M68000 - Genesis, Mac Classic, Amiga, etc.
* SM83 - GameBoy/Color
* ARM7TDMI - GBA
* SPC700 - SNES sound processor
* Z80 - ZX Spectrum, Galaksija, SMS/GG, Genesis sound processor, and so much more...
* SH4 - Dreamcast

With more to come....

With all that said, I did SNES was a long time ago. I may miss a thing here or there. Feel free to let me know.

### Order
I'm thinking I'll tackle CPU, then GPU, then APU, roughly. There are many more sub-components. We'll see as we go! 