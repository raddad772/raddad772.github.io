## JavaScript: Return of the Woes

—

### Intro
The day is here: Super Mario World and Link to the Past are both playing on my pure-JavaScript, cycle-accurate SNES emulator at full speed, even on lower to mid-range hardware. 

But the journey here wasn’t easy. Even getting any output from a basic test ROM required implementing most of a very complex system correctly. 

### Mo’ PPU mo’ problems

SNES needs a lot of code to work correctly before you can get even the most basic results. The console’s main CPU may be underwhelming, but they made up for it by adding DMA and taking the NES PPU and making it Super. 

Up to 4 independently scrolling background layers, with 2-, 4-, and 8-bit per pixel modes as well as direct color mode, configurable blending between them including translucency, the famous mode7, which used math to generate dazzling rotation and zoom effects. So many colors on-screen, with HDMA making it a relative breeze to use windows and color math to do big, full-screen effects.

You need to implement hundreds of registers that do fairly complex unintuitive things to layer visibility and color math in order to compose a scene.

I wrote my initial PPU implementation by following along the bsnes fast-PPU, because realistically, between all of the myriad systems interacting inside the SNES, I found it a little daunting. I’d already poured months of effort in and needed something to verify if DMA, SPC, etc. were all functioning correctly. 

After chasing down bugs, I ended up with a great image! And, unfortunately, not-so-great performance. 

### How embarrassingly slow!
On my mid-to-high-end work computer, I could run Mario World at about 27-30fps. It was super cool to have brought it that far, but super disappointing I was at around half the speed I needed to be at. 

However, the PPU was taking up 80 percent of execution time. Without the PPU drawing, the emulator simulated along at over 120 fps pretty easily. Surely there must be something I could do to take advantage of those extra cores that are just idling and speed up this embarrassingly slow process. 

Wait, that’s the wrong term. The one I’m looking for is “embarrassingly parallel,” which is a technical term that is used when a problem is very easy to parallelize. Often this occurs when elements of a data set you need to process are not interdependent.

Like pixels on a screen.

So at this point in development, the emulator is entirely single-threaded, despite the fact that it’s running 4 different emulations spread across 3 physical chips all clocked at different speeds. Thankfully, I designed it from the beginning  to allow for multithreading. 

### Knowing when you can use multithreading is half the battle
The less interdependent two systems are with each other, the easier it is to take advantage of parallel processing.

In the case of the SNES, we have these components to worry about:

* The R5A22 CPU, which contains some somewhat powerful DMA and HDMA circuitry, IRQ timers, WRAM refresh logic, a multiply/divide ALU, and the main 65816 CPU core. It has access to the ROM, 128k of work RAM, and uses memory-mapped registers to access the sound and PPU 
* The SPC700 CPU from Sony, whose main responsibility is to take commands and data from the main CPU and manage the sound mixing DSP. It has 64k of its own memory the main CPU can’t access except through a program you write.
* The PPU, which has multiple RAM modules, separate from the ones accessible by the CPU directly,  that it uses to draw the screen. It also has a few hundred configuration registers exposed directly to the CPU via the secondary (“B”) bus. 

What makes it possible to parallelize these is their relatively loose coupling and restrictions on their usage.

Dispatch in JSMoo worked like this, before I split out the PPU:

1. Determine the timing of the next scanline. Scanlines can be different lengths and have events happen at certain points along them, such as IRQ, NMI, WRAM refresh, etc. 
2. Once we know how many cycles will be in it, execute the scanline from the R5A22. 
3. The R5A22 will determine what to do, which can be do DMA or HDMA transfers, raise or lower interrupt signals, refresh WRAM, or run the 65816 core and ALU functions.
4. These are run in blocks determined using a scheduling algorithm that takes into account when things need to happen, to minimize the number of times this complex logic needs to be hit and maximize time spent emulating. It’s precalculated as much as possible and on most scanlines, it’s only evaluated 2-5 times. 
5. As a block is executed, a special function is called letting it know how far to move forward the system clock. This function is also called if a synchronization event occurs.
6. That function says, OK, the R5A22 has emulated 12 master cycles. Do we need to draw a scanline and schedule the next one and make sure the coprocessors are up to date?
7. After any synchronization that needs to happen, R5A22 execution continues, “driving” everything.
8. If a read from or write to the SPC700 happens, the R5A22 is paused while the SPC700 is emulated to catch up to that point in time. The SPC also catches up at the end of every scanline. Since the 5A22 “drives” the master clock, we don’t need to worry about a lot of synchronization pitfalls.
9. At the end of each scanline, the PPU draws the next one.

As you can see, under this paradigm, it is the R5A22 driving everything, and scanlines are drawn right as they are ready. 

But like I said earlier, the PPU is embarrassingly parallel. During the drawing of a frame, you are guaranteed that VRAM, CGRAM, and OAM won’t change, in every game except one. (Also, SuperFX games use fblank to get more DMA time, but this scheme works out for that, since the lines are blank and nothing is drawn, it does not need the update VRAM). So for all 224-240 lines of the picture, we know we could use a copy of these we obtained at the start of line 0.

However, any of the hundreds of PPU registers that control anything from default background color, to screen mode, to scroll positions, can be modified automatically with HDMA in between scanlines. So if we want to parallelize the PPU and make it possible for the timing of drawing to be decoupled from the emulation of the processor, we need a copy of all those registers for each scanline. 

### JavaScript is bad at that specific thing though
Specifically, it’s very slow about copying objects. To copy a nested object with a few hundred total members each frame, it ended up using around 20-30 percent of the execution time!

If you ever find yourself in that situation, I recommend you do what I did: take advantage of code generation. I wrote a function that inspects the properties of an object, and auto-generate a copy function that will copy that object member by member, which is so fast it doesn’t even appear in the profiler anymore for a huge speed increase. The final code looks like this:

This approach was later re-used to serialize and deserialize to shared memory buffers quickly. 

### Ok ok, so that’s all interesting, but how do you use threads in JavaScript?
The answer to that question is, JavaScript has its own way of doing things, which is different from everyone else, because JavaScript.

Instead of traditional threads, which you launch and are active until they finish execution, you get Web Workers. That’s kind of like a thread that doesn’t execute until it gets a message telling it to process some data. Or at least, that’s how it appears to the programmer.

Web workers communicate with the main thread and vice by passing messages. This is one popular and…fine…way to manage thread communications, but it was weird to me how they lock you in to only using that model.

Further, threads execute in an entirely isolated scope. They don’t even have access to class or function definitions from the main thread. If you want a thread to do something, you’ve got to put it in that thread.

Well, that’s not exactly true. You can import things like normal. But, I was forced to use a workaround for JavaScript denying access to local files for use as threads, and it forced me into that situation. 

On to he bug. Those messages will not be processed while a thread is currently running. I was used to other languages where if you call a member in a shared class, it will run on the calling thread, which is a useful idiom for doing things like indicating thread work completion. When it was time to draw a new frame, If the last one wasn’t done yet, I was busy-spinning waiting for a lock to be cleared by another function in that thread. It took me a little while to figure out why that wouldn’t work, and why a different thread had to clear it. 

Side note: the main thread cannot sleep, because the language designers think that can cause a deadlock. They don’t care about the hundreds of other ways to cause deadlocks, so much they make you implement your own mutexes, locks, etc. they don’t care how much more efficient it might be for other threads to use a core while it’s waiting. It honestly baffles me. 


### Actually drawing like a PPU

So now that I had multi-threading going, it was still super slow. I had to figure out how to do fast copies of chunks of RAM (which is actually a thing thankfully), and so as little automated copying as possible, because JavaScript is terrible at that. Unbelievably slow. 

So I took my PPU classes, and moved the code out to be more C-style. This mostly consisted of moving functions out of classes and adding any members they need as arguments until all the functions needed to draw the screen are isolated. Then copy to worker and we’re golden! Except it’s messy and I’m copy-pasting code because JavaScript. But what can you do?

### One last frustration

I had split up the code, gotten it working correctly and displaying a picture, optimized the heck out of memory and object copies, and I was still waiting…33+ milliseconds for a single frame to draw? What’s going on here?

JavaScript does not give you high-resolution timers. They are guaranteed to be about +/- 4 milliseconds, to guard against timing attacks. When you’re looking for steady 16ms frame pacing, that’s a 25% variance and completely unacceptable.

The only mechanism I could find to smoothly animate is a function called requestAnimationFrame. That is the browser requesting animation from your program. This will call your code (supposedly) at the refresh rate of your monitor, and provide you with an accurate count of how long since the last frame. Doesn’t that sound perfect?

Just one catch. Chrome, at the least, will throttle this, seemingly arbitrarily. Too many Chrome windows open? It only gets called at 30fps. Power saving mode? 48fps. Moon in the wrong phase? 30fps.

Further, neither the documentation nor the SDK tells you it’s doing this. You’re meant to advance your animation by the elapsed time, but the timer it gives you doesn’t tell you if your code is taking 35ms, or if it only took 10ms and waited 25ms for vsync. 

It took me 3 hours of frustration to figure out it was doing this. Finally disabled emulation and counted calls per second of that function.

After restarting my computer (?), everything ran at 60FPS smooth as butter. 

Since then I’ve noticed it doing this all the time, and I’m looking for an alternate mechanism to time things. But when Chrome allows it for reasons it doesn’t tell you, the emulator now works great!

### Summary
I really wouldn’t recommend JavaScript for a complex (above NES level) emulator unless you want a real challenge. You’re fighting a poorly designed language, trying to code your code so it is optimized by the JIT well, dealing with untyped numbers, and also fighting a lot of different security mechanisms put in place because you’re running without trust from the browser. Emulator development is already hard enough without all these headaches. 

JavaScript object/class copies are very very slow, avoid them at all costs, and generate code to do it for you manually

Timing is a pain in JavaScript. Be aware that even the way everyone recommends to time won’t always fire at 60 fps. You have no control over this and it does not indicate to you that it is doing this.

Finally, threads are useful, but JavaScript is a pain with them. And if you want to use real shared memory instead of copying everything (already understood to be very slow), you need to be served from a specially configured web server that provides the browser with specific headers. 

But it is doable! Persevere and you’ll prove those people who say it’s not possible in JavaScript (or, in the case of the SNES, not possible at all) wrong!

### What’s next for JSMoo
There’s a long list of to-do’s, from GLSL PPU drawing, sound output, enhancement chips, UI improvements, sage states, better debugging, maybe more systems…

Sometimes, working months and meeting your personal goals can be a little bit of a letdown. You accomplished what you set out to do, now what do you do?

Now that I’ve proved it possible, I think I’d like to take it all the way. I like the idea of having a bunch of emulators freely accessible and able to be learned from by anyone with a browser. I don’t know if I’ll ever get there, but I’d like JSMoo to have great SNES compatibility, and play other things too. And at this point I’m thinking PSX and perhaps N64 might be possible, too. 

First, though, I need to refactor. So much code was written in ignorance of how it would be mangled to make it behave properly, because I did not understand the systems yet. It’s ugly and full of pointless comments speculating about things I know now. I’m going to make it better before I build on it too much.

Thanks for reading so far!
