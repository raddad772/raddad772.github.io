## Testing your CPU emulator

And a bit about emulating in the first place
---

This post is a work-in-progress

### Test Vectors Needed
In my last post, I detailed why (I believe) the 65C816 wasn't more popular, despite being a 16-bit extension to one of the most popular processors in the world at the time.

This time, I'll be writing about testing your CPU emulator.

If you're writing a GameBoy (Z80) or NES (6502) or many arcade (Intel 8080), etc. emulators, you're in a pretty good spot. You've got a TON of other emulator source codes to look at. You've got really good documentation. You've got test ROMs specifically made to verify how your instructions are doing.

If you're emulating an SNES or Apple IIGS, you have two main emulators to check out, both of which are readable but not exactly precise. What I mean by this is that it's not easy to deduce cycle timings from the Higan code, for instance.

And the sorts of test ROM I want do exist...for the SNES. They're usually more focused on ensuring the whole SNES emulator is accurate, but ones do focus on the individual instructions. To actually use one, you need the convoluted memory mapping scheme, DMA and HDMA, at least basic PPU emulation, etc., though.

I'd like to verify my 65c816 core a long time before I get all that going. There's no sense debugging my memory mapping code when I don't even know if the input addresses are likely to be any good. So after looking around, I decided to write my own.

### What are we even testing?

The fun thing about the 65c816 is that testing it is actually pretty straightforward. There's 256 opcodes, but you don't actually have to individually test all 256 to gain some certainty you're doing it right.

There are really two ways to "slice" the instruction set: into Operations, such as ADC (Add with Carry), ROL (Rotate Left), etc., and Addressing Modes, such as Directed Indexed Indirect.

If that sentence doesn't make any sense to you, check out my previous post where I go into detail about this.

### Testing instructions
For instance to test LDA, STA, etc. only need to test different R, W, RMW versions...

### Programs, assemble!
Writing an assembler

### Environmental issues
Creating a simple, fake computer for it to run on, that other people may be able to reuse some day on their own emulators

### Example code

### Next post
Adventures in optimizing 65c816 code