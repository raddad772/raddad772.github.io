## Testing your CPU emulator

--when it's not very popular
---

### Test Vectors Needed
In my last post, I went into a bunch of technical stuff about how and why the 65c816 is a pain to emulate on its own merits, and my solutions for it. I've also mentioned previously how the 65c816 was not a popular processor, which is, I believe, because there were much better alternatives available.

All this means that there's basically only two meaningfully different SNES emulators to look at, and not a lot of test vectors available. You want to test your 8080, Z80, 6502, or even 68000 core? There's repositories filled with thousands of tests.

You want to test your 65c816 core? Well, you better have most of a functional SNES emulator on top of it, because there isn't really much that runs on less.

I'd like to verify my 65c816 core a long time before I get all that going. There's no sense debugging my memory mapping code when I don't even know if the input addresses are likely to be any good. So after looking around, I decided to write my own.


### Environmental issues

Of course, I don't want to contribute to the paucity problem. I want my code to be able to (a) be itself easily verified on another emulator, such as the 65c816 core from Higan, and (b) be useful for future emulator authors.

So I designed a very, very simple environment for it to execute in. I could have taken the shortcut of outputting text after a BRK instruction, or using a character-mapped display so you just have to write strings where you want them in memory, or some such, but that's honestly not that cool. I also want a bit of a fun challenge actually writing this code.

What I went for is a very simple computer. It runs 60FPS, 600kHz, but that's pretty arbitrary. It has a 256x256 monochrome bitmap display, taking up all addresses at bank $01. A 0 byte is black, any other byte is white. This is an opportunity for future expansion, because you can make a color display out of this and have backward-compatible programs. It's also a challenge, because there's NOT a lot of cycles to work with per frame, and working with data in multiple different memory banks on the 65c816 with any speed is kind of a pain.

Also, I have plans for basic input, if it's every needed, but that's outside my current scope.

I mostly want to emulate that CHIP-8 ROM which shows on-screen which instructions are working correctly. That's the ultimate goal of this test program, to give an emulator author a simple environment that can get some kind of graphical results and also good feedback on their emulator.

But as we covered in the last post, there's 1,280 - let's say closer to 800 if you reduce it carefully - different opcodes, spread across 90+ instructions and 45+ addressing modes, to test. No way am I going to spend the time to write a test ROM with every possible instruction.

### Easy65c816!
One of the coolest retro projects I have ever seen is [Easy6502JS](https://skilldrick.github.io/easy6502/). It combines an eBook intro to coding the 6502 with an inline assembler/disassembler/simulator, in JavaScript, right in your browser. Go check it out! Isn't it cool?

While researching 65c816 assemblers, I found nothing like that. There's an assembler online, but it's not easy, nor does it come with an execution environment.

So I decided to write my own, inspired by Easy6502. It's not quite as feature complete yet, but it takes 65c816 assembly programs that mostly follow the WDC manual specifications - plus a few additions of my own - and creates a ROM for you. It then runs it. It isn't all interactive and great like Easy6502 is yet, and also still has bugs, and the source is complete garbage because I didn't really want to write it and did it all in one short go with no planning. But it's on the way toward being like Easy6502. I just need to get off my lazy bottom and write the JavaScript portion of the interface.

So, now I have my tools to actually create the ROM. But as I mentioned above, there's a LOT to test! Is there some way to narrow it down?

### What are we even testing?
The fun thing about the 65c816 is that testing it is actually pretty straightforward. There's 256 opcodes and lots of variations due to operating modes, but you don't actually have to individually test all 256 to gain some certainty you're doing it right.

There are really two ways to "slice" the instruction set: into Operations, such as ADC (Add with Carry), ROL (Rotate Left), etc., and Addressing Modes, such as Directed Indexed Indirect.

If you've implemented each variant of each addressing mode correctly, then it should be correct with every single instruction. And likewise, if you implement your ADC8, ADC16, ADC8-Decimal, and ADC16-Decimal correct, it shouldn't matter the addressing mode it is operating on.

Verifying each addressing mode in-depth, and each instruction's operation in-depth, should get you most of the way there; you can hit two birds with one stone in many cases, and verify an addressing mode along with an instruction in the same test case. Or at least, that's my theory.

### A quirky architecture
Buuuut, this is really only the start of the quirks. There's instructions that act differently between Emulation mode and non-Emulation mode just as you'd expect, like Push A. In Emulation mode, it will of course only push 8 bits, but the Stack pointer will wrap around and stay in the 0x000100-0x0001FF range. Except for on some instructions, in which they will write outside that range, but then wrap the stack pointer after (basically messing up your stack). There's all sorts of caveats like this, and I honestly haven't implemented htem all properly, but htat's what the test ROM is for.

### Testing is important in general
I'm not doing all this for posterity, though. It's important to have thorough tests in most software systems. How can you be confident that a bug you fix won't cause other bugs? Tests. How can you be confident that a timing change you make to fix a game won't break other things? Tests. Tests are great ways of making your code more maintainable, and having a test program to run each time I make a change will be a great aid in debugging the frankly annoyingly complicated aspects of the SNES.

### OK OK, what does all this look like?
Well, I haven't gotten the ROM written yet. Just getting "Hello, world!" and any other arbitrary ASCII string to print onto the display at abitrary coordinates was a learning experience. I also felt the need to optimize my initial attempt, which took ~180k cycles, as much as I could - it ended up at about ~25k cycles. And I found a bunch of bugs in both the assembler and the CPU core just doing that.

I plan for my next blog post to be about that optimization process, just to give you a taste of what programming the 65c816 looks like, and what people had to go through to write performant code on this thing.

Thanks for reading! And I hope you don't find all my complaining discouraging. I definitely would not recommend the SNES or the 65c816 as a first or even second emulation project, but for those with the passion, these are all just a bunch of fun puzzles to solve.
