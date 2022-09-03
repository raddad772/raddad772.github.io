## Timing for games and emulators in JavaScript, and, a tiny bit of news

—

### First, a little news
Timing in JavaScript is hard, because of all the protections that vendors put in against Spectre, and also because some of the design decisions the language makers have made.

I didn't want to tackle the difficult problem by trying to synchronize SNES, because SNES itself is a very complex system, and I'm running 8+ threads for it. So, I added NES support!

Of course, I couldn't just do boring high-level support. I had to go and emulate individual PPU cycles and the specific way it pre-fetches sprites and does memory accesses to draw pixel-by-pixel with 100% accuracy. This is great for compatability, but it took a while to get right.

I got the 65C02 core up to speed (for planned Commodore64), and I'm working on SMS with similar accuracy right now. Then I'll tackle sound synch. But my NES core was mostly complete, so it was time to get into...timing!


### Intro
We have two very simple problems we need to solve, and both revolve around timing.

### The sound problem
How do we get sound sent without crackling? No idea yet, that'll be a future post.

### The frame-rate problem
How do we get our emulator to go 60FPS? Not 40FPS, not 80FPs, but 60?

The conventional wisdom is to use requestAnimationFrame() callbacks. This is a function provided by the browser which, ostensibly, will call your function of choice once for every update of your monitor. Got a 60Hz monitor? Perfect, just hook it up to requestAnimationFrame() and you're all set!

Except when you're not.

See, there are a few problems with this:

 - If the user is on a laptop or mobile, device, it may spuriously decide to "save power" by bringing down the number of requested frames. This is not documented, but I have confirmed it across several devices on Chrome. You performance gets a fun pie chart where you're only getting 48FPS, and over 40% of your time is spent idling. What frustration!
 - If the user has a screen with a refresh rate that isn't 60Hz, it's a pain to detect and work around this
 - If you want to implement a "fast forward"-style button into your emulator, too bad for you, you get what you get. There's obviously ways around this, it's just that this timing is so inflexible
 - Sometimes, for unknown reasons, Chrome will just throttle the number of requestAnimationFrame() calls seemingly at random. This happened for me when I implemented line A12-watching on my NES. This is a bit expensive, but still well under 2% of total execution time. Still, it pushed my NES core from 60FPS to 40FPS (which is a big ? for me)
 
 All in all, the conventional wisdom isn't really that great if you want an accurate timing system.
 
 ### Why not just use timers like a C++ programmer would?
 Good question! It's made harder by the fact that, due to the threat of Spectre, browsers make timers have a resolution of close to ~4ms. It's possible to use...WAYS...to get around this.
 
 BUT.
 
 We're using Web Workers for multi-threading already, and we're using SharedArrayBuffers()'s to cut down on the amount of data being (very slowly) copied by JavaScript. In order to enable those, you have to modify the HTML headers on your website.
 
 This, unfortunately, is taking us out of the realm of something that can be just loaded off your disk as .HTML file and run. However, we're already passed that. So we have access to performance.now() at <.1ms accuracy.
 
 It's not quite as useful as it could be, because timestamps' 0 starts when the thread does. This means that they cannot be compared across threads precicesly. But perhaps if we dedicate a thread just to timing...
 
 ### A dedicated thread just for timing
 In JavaScript multi-threading, you have a big difference between your main thread and web workers.
 
 Your main thread gets access to all the APIs, DOM, etc. On the other hand, it has no way to "sleep", because the developers think it's too easy to cause a deadlock with a sleep. (The only sleep functionality threads have is Atomics.wait, but that's another story). I find this a little ridiculous, since it's super easy to cause a deadlock a hundred other ways, and I would rather my language treated me like an adult, but whatever.
 
 Threads have a very restricted subset of features they have access to. They live in their own namespace, as a .js that includes other .js files they need.
 
 Once instantiated, a thread is actually always sleeping, waiting on a message from the main thread. Threads can send messages back to the main thread, too. These messages will be placed in a queue until the current operation completes. Messages will NOT pre-empt the running thread, and the common paradigm of calling a class's "handle_message"-type function from the calling thread, knowing it'll execute there, is not possible in JavaScript.
 
 So with these caveats in mind, I designed a class and a worker: timing_header.js for use by the main thread, and timing_worker.js to do the work of a timer.
 
 ### Basic architecture
 The timing class can be given any arbitrary FPS to try to hit, and be told to play and pause.

It works like this. Let's say we have a 60FPS target frame rate, so 16.66666666667ms target frame time.

Main : Send play message, end execution
Timer: Request frame via message, end execution
Main : Render frame, send message to timer that frame is rendered, end execution
Timer: Measure time from requesting frame to being told it is complete. Perhaps it has been 10ms. So we must wait 6.666666666666667ms before requesting next frame via message and ending execution

That way, the main thread only needs to worry about sending "pause," "play," and emulating a frame when it's requested; the timer worries about pacing the frames.

That all sounds easy, right? It would be!

### How do we wait?
In Computer Science, there's generallys two ways to wait: a busy-wait, where we're checking a timer over and over and over, and a sleep method, which allows other threads to use the CPU execution resources while we're asleep.
By default, PCs schedule threads in 1ms slices anyway, so you'll get pretty close to your 16.67ms if you just measure frame time and sleep the rest.

Unless, of course, you're using JavaScript.

The wait-like procedure is Atomics.wait(). To use this, you need a ShareArrayBuffer, along with an index inside it that won't change. You're basically telling it to wait for a value to change, and time out if it doesn't. So if you want to sleep 4ms, you can say, Atomics.wait(SAB, 0, 4). Your execution SHOULD resume in 4ms.

In my testing, about 90% of the time, it works exactly as intended. About 10% of the time, it randomly takes 1-4?ms longer for no discernable reason.
The best way to wait is usually some kind of sleep() method.

You may be wondering, why not use setTimeout()? In my testing, it was not very precise either.

This leaves us with the rather distasteful prospect of using a busy loop, which looks like this:

```
let i = 0;
while((performance.now() - this.frame_start) < this.frame_time_target) {
	i++;
}
```

Tying up potentially a whole CPU core just adding 1 to a variable over and over (so the loop doesn't get optimized out) until we need to be done waiting.

Since this is technically time the emulator isn't emulating anyway, that would probably be OK, but it still smells bad. So I decided on a two-step method to wait.

Step 1: Atomics.sleep() up to 80% of the time we're supposed to wait before requesting a frame. This way if we wake up a little late it'll probably be OK, and we're not just busy-spinning constantly.
Step 2: Busy-spin to 90% of the way. This lets us start the next frame a little early, in case it needs slightly more than its alotted time, without causing too much input jitter. If you wanted to be more precise, setting this to 99% works just as well.

And this way of doing things worked! NOT!

### Always re-check your assumptions
I got an incredibly steady 40FPS with this timer code. As I put the target frame-rate toward 1000, I gradually approached 60FPS, until it hitched up into the 90's randomly. I'm ashamed that I spent a few hours finding the cause of this.

Let me start by saying, I implemented NES in order to test the timing code (and am going in this direction for the sound synch code, too) because it's a lot simpler and more predictable than the SNES code. So now, the emulator runs a few SNES games and most non-mapper and MMC3 NES games.

Anyway, I took a look at my frame-to-frame times. My numbers looked like this:

25ms
15ms
3ms
26ms
14ms
1ms

See a pattern? I did, but couldn't make sense of it. Extra-long frames might make sense, but 1ms frames? I spent a bunch of time figuring out why my frames were so inconsistent. Were messages being delayed randomly? Were frames being skipped? What was going on!?

I had to check my assumptions. My run_frame() function looked like this for the NES:

```
run_frame() {
	let lines_to_do = (this.clock.timing.frame_lines - this.clock.ppu_y);
	for (let i = 0; i < lines_to_do; i++) {
		this.run_scanline();
		if (dbg.do_break) break;
	}
}
```

That should work fine, right? But when I finally put a console.log in there to check on how many lines were actually getting executed each request...

262
1
262
262
1
262

Apparently sometimes my timing code had 263-line frames. That's a bug to investigate, but honestly, I'm glad the only one I had was this simple, and not a subtle thing deep in the logic.

New function:

```
run_frame() {
	let current_frame = this.clock.master_frame;
	while (this.clock.master_frame === current_frame) {
		this.run_scanline();
		if (dbg.do_break) break;
	}
	//this.ppu.render_bgtables_from_memory(0, 260, true);
	//this.ppu.render_chr_tables_from_memory(0, 260*3);
}
```

Now the timing code works nearly prefectly, maintaining 58-62 FPS (and mostly just 60), so yay!