## The journey so far: what I've emulated

I started this emulation journey a good 5 years ago at this point. I've come a long way, learned a lot, and gotten to give back to the community as well. I figured maybe it's time to reflect on that a little.

### Initial motivations
Everyone gets started in emulation for their own reasons. Here are mine:

#### Fibers are slow
I was reading an article by Near a ways back: 
[Accuracy takes power: one man's 3GHz quest to build a perfect SNES emulator](https://arstechnica.com/gaming/2011/08/accuracy-takes-power-one-mans-3ghz-quest-to-build-a-perfect-snes-emulator/)

I went and idly looked at the source to Higan, and noticed that it made extensive use of fibers.

Fibers are a co-processing construct that allows you to suspend a fiber of execution partway through inside a function, and pick it up partway through. It's like Python or Rust generators, or coroutines. Near had a C++ library for it.

However, from a theoretical perspective, fibers are slow. They (at least theoretically) have all the overhead of threads: you need a call stack, local storage, context switching by pushing and popping registers, and more.

Considering the cycle-accurate approach taken for SNES, I figured that it could actually all be done, a lot faster, without fibers. And I wanted to prove it.

As a bit of a contrast, fast and accurate MAME cores sometimes use a more explicit form of this, where a function will attempt to complete. If it can't, it will leave the info needed for a helper function to pickup partway through. This leads to some pretty ugly and gnarly code, but it's fast.

Another option would be to use lots of case statements and explicit state. This is the way I chose to go.

I had the notion that, by structuring your emulator correctly and using the right techniques, you could get all the accuracy, along with the speed. Please note that this is not an attack on Near or bsnes/Higan/Ares (bsnes evolved into Higan, which then evolved into the current emulator, Ares). I just disagreed with a design decision and wanted to prove my point, that's all. 

#### Initial motivations, part B: build rot
If there's one thing I hate, it's going to experiment with another emulator and finding out I can't build it. The libraries are no longer available, or my platform isn't supported, or it won't work on newer compilers, or, or, or. This is called build rot and if you've ever tried to build a project that hasn't been maintained for a few decades, you'll probably know the pain.

In an attempt to make easily accessible, accurate emulators that are easy to build and experiment with for anyone, and to more fully prove my point about the architecture of an emulator, I decided to focus on JavaScript. Theoretically, a webpage should experience less build rot, and old web browsers are easily available to run things on nowadays. I hoped it would remain that way.

### The journey
So I started an SNES emulator. I named to JSMoo because of a reason that only makes sense to me. Emu is the name of a bird, and they kinda remind me of cows sometimes, and cows go moo, and moo kinda sounds like emu. Add JavaScript to Moo and you [JSMoo](https://github.com/raddad772/jsmoo).

Note: Although the source is helpful to many, I rarely put any work into the UI, making it hard to use and frankly annoying. I always planned to do the UI later, just never got around to it before I switched away from web technologies altogether.

I worked on the 65816 CPU, and found [TomHarte's excellent tests](https://github.com/SingleStepTests/65816), which I helped him refine while fixing up my 65816 emulator. Then I started work on the rest of the SNES hardware.

I did original work on the 65816, the scheduler, the SPC700 (which I eventually made my own JSON tests for), the multi-threading of the PPU (a subject of a previous post, "PPU Woes in JavaScript"). The PPU code itself was largely from bsnes-ppu-fast, and some of the other hardware code was very heavily sourced from bsnes. At the time I wasn't really thinking about releasing the emulator or anything, I was just trying to work on that point. If I ever port this emulator in the future I'll re-do large parts of it so it's more original work.

Regardless, I proved to my own satisfaction that it was possible to get high accuracy with better speed with a different architecture. Some people dispute this because when they say accuracy, they mean, % of working games (comptability). For me, I was trying to show that you could simulate the hardware with the same granularity (cycle accuracy).

### Other emulators
I soon moved on to other emulators. In JavaScript, the final tally of systems was:

* SNES
* NES
* GameBoy/Color
* Sega Master System/GameGear
* ZX Spectrum
* PlayStation1 (kindof)

And as for .JSON tests, I generated the following:

* Z80, from a JavaScript port of parts of Ares' Z80 core, which then was bugfixed and had its accuracy increased thanks to members of the community (and I made relevant Issues to Ares). This was made because I couldn't get my Z80 cores to run anything.
* SPC700, the sound co-processor inside the SNES, which used a partial JavaScript port of Ares' SPC700 core. This was made to help my own SPC700 core
* SM83, the processor inside the GameBoy, by again making a partial JavaScript port of Ares' SM83 core.

However, PlayStation on JavaScript discouraged me. See, JavaScript has some issues with integers. You pretty much can only have 32-bit signed integers. If you want bigger, you need to use BigInt, which is about an order of magnitude slower. You can certainly get around this in JavaScript, but I didn't want to.

### WebAssembly/AssemblyScript
I wanted to keep the easy-to-use-and-modify nature, so settled on WebAssembly, which had 32-bit unsigned and 64-bit signed and unsigned integer types.

Looking for a good language to use with WebAssembly, I came across (and liked) [AssemblyScript](https://www.assemblyscript.org). It's a cool language for what it is. It's based on TypeScript, so it should be easy to port JS emulators to it. I ported NES, GameBoy (not color), SMS/GG, and ps1 to it. I developed the PS1 emulator to the point it could boot and run some homebrew games.

However, I found its multi-threading support to be lacking. Also, only Chrome really ran it well, and I much prefer open source and free software such as Firefox. After casting around some, I decided, hey, let's just do this all in C.

### C
C because: 40-year-old C programs can be (and still are) built from source. As long as you conform to a standard, it's not TOO hard to make things build. Most people can understand C, unlike Zig or Rust (which I have a problem with). Finally, it can compile to webassembly as well

However, I recognized that UIs are not the same, and will almost inevitably suffer from more extensive build rot. I decided to split the project into 3:

* jsmooch-tests, a small testing project for pieces of the emulator.
* jsmooch-gui, the GUI portion, still written in C (now in C++ for ImgUI)
* jsmooch-lib, the actual emulator cores, written purely in C11-compliant C.

JSMooCh, you ask? Well, take JSMoo, add C, and you almost have "smooch," so I just made it JSMooCh. It's really not a lot of thought put into it.

### Since then...
Since that time, I've ported and minorly enhanced these from JavaScript:

* NES
* GB/C
* SMS/GG
* ZX Spectrum

I've also added:
* poor Atari 2600
* Not-yet-functional Sega Genesis
* Semi-functional Macintosh Classic
* WIP Apple IIe
* Sega Dreamcast (right up to where video commands start happening)


I've also created single-step JSON tests for 
* SH4, the processor inside the Dreamcast, by hacking up Reicast
* The Motorola 68000, the processor inside the Sega Genesis and Mac Classic (and Amiga and more...), by hacking up Olivier Galibert microcode-based, mostly-cycle-accurate MAME M68000 core

I've also made steps toward a halfway-decent GUI, with a implementation-dependent in-depth debugging suite.

I'm planning on doing .JSON tests for ARM7TDMI using NanoBoyAdvance, and MIPS r3000 (PS1) with I-don't-know-which-yet emulator. 

### OK

That's a summary of my emulation journey so far. I really have liked working in pure C11, though I do miss some conveniences of C++. 

As a final bit, I'll list the CPUs I've emulated so far, in order of how fond of them I am:

* Motorola 6502, 65c02 (Commodore64, NES, atari, apple II, so many more...)
* Hitachi SH4 (Dreamcast)
* WDC 65816 (SNES)
* Sony SPC700 (SNES)
* LG SM83 (GameBoy/Color)
* MIPS r3000 (PS1)
* Zilog Z80 (SMS/GG, ZX Spectrum, and many more)
* Motorola 68000 (Amiga, Sega Genesis, Mac Classic, so many more...)