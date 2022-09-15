## Unit tests for your CPU, and the weird, wonderful ZX Spectrum

—

### First, a little news
#### TL;DR
I added ZX Spectrum 48K support and unit tests for the Z80.

#### Longer version
ZX Spectrum 48K is working 100%! Except you can't really load any ROMs or tape files or anything, and a few keys don't register properly, which is purely a UI issue. So like...97%?

But it boots and you can use BASIC. Awesome, my first computer is up and going!

But why the ol' Speccy? Am I British or something?

No. I'm American. But the ZX spectrum has two important things going for it:

1. It uses the Z80, which I wanted to make an emulator for in order to get to my next goal of Sega Master System emulation
2. Its hardware is simpler than I thought possible.

It's so cool and simple how it works, I thought I'd make a blog post about it!

If you don't have unit tests like the ones Tomas Harte makes available, when you write an emulator, you're writing tens of thousands of LOC, without really being able to test it.

Before you can get any inkling if your code is performing, you not only need to emulate a CPU, which is a pretty big task, but also, at the least, some input, output, and basic peripherals.

I decided to work on the Speccy because it's so simple, I'd have to write less code than the SMS before I could test the CPU.

### Did that work out for me?
No. No, it did not work out for me.

As I'll get to in a moment, the Spectrum's hardware is amazingly simple...but the OS is not. I found myself stepping through it over long periods, double-checking the execution to find bugs in my Z80 core. It was a slog. So I decided to just write the unit tests.

I already wrote a unit test generator for the SPC700, so I built off that. I used Ares as a reference Z80 implementation, Frankensteining parts of its Z80 core into JavaScript to generate the tests.

I verified my core against them, fixed a bunch of things, and the Speccy was working. Great!

Both sets of tests are available inside JSMoo, in the misc/tests/GeneratedTests folder. The root for the source is in misc/code_generation, and you can generate the tests yourself with a bunch of different options.

### OK so what's all this "weird, wonderful" stuff?
If you're American, and are asked to think about which home computers made the biggest splash in the early 80's, you'll probably think of the Commodore64 or an Apple II. Both are great computers that have a ton of features. Both are actually fairly complicated to emulate.

The Commodore64 is a 6502-based computer with 64k of RAM in a 64k address space. It accomplishes this by switching different memory-mapped I/O, ROM, and RAM banks in and out all over the place. It has a complicated graphics chip with multiple screen modes that can do hardware sprites, and a really complicated audio chip that still sounds pretty good.

The Commodore64 isn't really that unusual, though. In Britian, they had the BBC Micro, another 6502-based computer with fairly similar features and complications.

The thing is, the Commodore64, BBC Micro, and ZX Spectrum were all released within months of each other, and the Speccy was...very basic.

The base model came with just 16K of ROM, 16K of RAM, and a built-in keyboard. The upgraded model had 48K of RAM.

There's absolutely no complications. It can read from the keyboard, read/write an audio port (to read/write audio tapes for data), and output very basic, 256x192 2-color bitmapped graphics. Which 2 colors are used can be chosen in 8x8-pixel blocks. And a speaker can beep.

And that's it. No complicated graphics modes, no weird timing (well, except for the way the ULA (grpahics chip) can pause the Z80 during certain memory accesses), no plethora of chips to emulate. You just need a 80, an EXTREMELY basic video output, a keyboard input, and almost nothing else. In fact, to emulate as much of the Speccy as I have, there's only about 570 lines of source code aside from the CPU emulator!

Contrast that to the NES, where I devoted 1,000 lines just for the graphics chip, or the SNES, which has nearly 2,000 for the graphics. This is not counting or considering the whole rest of the systems; the ZX Spectrum is a dead-simple piece of hardware.

I love clever old hardware like this. My goal with the SMS is to get Outrun running, because my wife loved it as a kid. Then I might just revisit the Speccy and flesh out the emulator. What a cool little piece of kit!