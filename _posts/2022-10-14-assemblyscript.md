## WebAssembly via AssemblyScript

### WebAssembly? What about pure JavaScript?
So, the emulators run great! On Chrome. Unfortunately, Firefox just isn't as miraculous at squeezing out performance. Emulators that run great in Chrome...do not, on Firefox.

Also, we could use a little more headroom on mobile devices.

So although I may keep developing emulators in JavaScript first (see below), the more feature-complete versions may end up in WebAssembly via AssemblyScript.

### Wait, what's WebAssembly?
WebAssembly is the "fourth language of the web" after HTML, CSS, and JavaScript. It's a binary bytecode similar to what processors execute, with the intention of executing at near-native speed.

Chrome already does a pretty good job at this, and the theoretical performance of a WebAssembly program isn't actually that much higher than a JavaScript one. But WebAssembly is consistent. It doesn't have the start-up time. And it should be fast on browsers OTHER than Chrome, too!

### Wait, what's AssemblyScript?
AssemblyScript is a language based on TypeScript, so it's pretty similar to JavaScript. It's meant specifically to compile to WebAssembly.

Since JSMoo uses so much code generation, theoretically, I should be able to re-target the code generation and produce AssemblyScript, which will then compile to WebAssembly. And the non-generated portions, I should largely be able to copy and paste and slap types onto.

### Some metrics
First. We want to make sure the finished product is doing as good as JavaScript. I haven't measured on FireFox yet (see below), but the NES emulator, on my PC, running Super Mario Bros. 1, pure JavaScript, single-threaded, produces one frame in 8.5-9.5ms.

How is SNES possible full-speed if NES uses so much time? Well, I emulated the internals of the NES PPU, including its memory access patterns and internal registers. This is extremely accurate, and very slow, in comparison to other methods, like just drawing from memory. The SNES currently has a cycle-accurate CPU emulator, an instruction-accurate SPC emulator, and a scanline-based PPU. The SNES also offloads the PPU drawing onto other threads. In comparison, the NES has a 100% cycle-accurate (minus bugs) CPU and PPU.

Anyway. 8.5-9.5ms is the number to try to reach or even, beat.

### JavaScript to AssemblyScript
I spent about 2-3 hours exploring AssemblyScript, 4-6 hours porting the NES emulation code (including re-targeting the M6502 core generator), and about 4-6 more hours integrating the NES core and getting a picture out of it.

Porting isn't 100% done. It currently can't read input, print processor traces, step, or a bunch of other functionality I need to work out. It just plays the game with no input right now.

Still, 10-15 hours of work to port an entire NES emulator to a new language and build system I'm not used to is pretty good. Especially considering that I also set it in its own separate thread, which accounts for at least half the time spent integrating it. 

### Why the separate thread?
I move the NES to a separate thread because I read up on how the main browser thread works and how unpredictable it is. A browser can take 1-12ms just to do tasks needed for the UI. In fact, iOS refers to the main thread as the "UI thread," and that makes great sense. If you're doing something like a game, mostly just the UI should go in the main thread.

The heavy lifting logic should be put in separate threads if at all possible. So I did that.

### Challenges with AssemblyScript
AssemblyScript is still a maturing platform. It's lacking some things that would be really nice, like bound-argument function pointers. It also has very poor support for threading.

Specifically, its inside-AssemblyScript threading requires you to use WebAssembly directly, and has some huge caveats with memory management. If you try to load it in a Web Worker like I did, you'll find that the way it automatically generates the glue code, you need to launch your Web Worker as a module. If you have something like a parsing error doing this, Chrome will just fail with no error, which is frustrating. And FireFox doesn't even support this yet!

You can write your own custom glue code, but I'm not comfy enough with it yet. Someone on the Discord mentioned using a different binding type too, which I haven't gotten around to yet.

So threading isn't easy. Moving data into and out of it is pretty easy if you don't mind copying it. If you want to do something fast liek share a frame-buffer, it's actually not too hard, if a little unintuitive.

### How to share memory between AssemblyScript and JavaScript
When you start up AssemblyScript, it gives you an object that represents the linear memory. You can pass out a pointer and access this memory.

Here's, roughly, how I shared the framebuffer from inside to outside.

First, to declare a buffer, instead of doing the normal thing and making a typed array, you need to use heap allocation.

```typescript
// Don't forget to manually free this later!
let ptr: usize = heap.alloc(size_to_allocate);

// Now we want to write to it, we do it like so...
store<u8>(ptr+index, value);
```

And in JavaScript, assuming we have passed the ptr out and it is available as 'ptr' to the JavaScript...

```javascript
// Establish a framebuffer via SharedArrayBuffer to share between threads
this.shared_array_buffer = new SharedArrayBuffer(however_big_you_want);
this.framebuffer = new Uint32Array(this.shared_array_buffer);

// .. later on, when we want to copy the framebuffer
//  out of AssemblyScript and into JavaScript...
// Where 'memory' is the object given to us by the glue code
//  that lets us access AssemblyScript memory.
let webassembly_buffer = new Uint32Array(memory.buffer); // View as Uint32's
let to_copy = however_many_uint32s_you_want;
this.framebuffer.set(webassembly_buffer.slice(ptr >>> 2, (ptr>>>2)+to_copy));
```

Once you have it figured out, it's not so hard. It's pretty speedy, too. Only one copy operation to get the finished framebuffer to the main thread for display!

### Ready, set, ...slow?
After solving some challenges, I booted it up and got a picture! Mario worked almost right away!

...Slowly.

I timed the code and found out that frames were taking about 25ms. That's 6ms longer than the speed they HAVE to be, and only 36% the speed of the native JavaScript code!

I was pretty disheartened at this point, but I headed into the AssemblyScript Discord for advice. Some nice people there looked my code over, showed me how to profile my code, and offered suggestions based on the output.

I found that the code was spending 3.6ms inside...Int32Array.get()!? What!? Why!?

### AssemblyScript TypedArrays are not like built-in C arrays
The TL;DR of the advice is that I should replace my TypedArrays like Int32Array with StaticArray<i32>, and get rid of scalar references like this:

```typescript
whatever: Int32Array = new Int32Array(4);

whatever[0];
whatever[1];

// should become
whatever0: i32 = 0;
whatever1: i32 = 0;
//  ...etc.
```

The change to StaticArrays is because Int32Arrays have an additional level of indirection that StaticArrays get rid of.

Furthermore, StaticArrays are performing bounds-checks. This is great for debug code, but when you're trying to run optimized code, taking 2+ms to check bounds you know are going to be OK is not OK.

Most compilers will let you disable bounds-checking with a flag, but in AssemblyScript, the code goes from this...

```typescrypt
let foo: Int32Array = new Int32Array(1024);

let bar: i32 = foo[10];
foo[0] = (bar + 4) << 4;
```

To this

```typescript
let foo: StaticArray<i32> = new StaticArray<i32>(1024);

let bar: i32 = unchecked(foo[10]);
unchecked(foo[0] = (bar + 4) << 4);
```

Just doing that for basically all my TypedArrays cut execution time per-frame down from nearly 25ms ... to about 10-11ms!

10-11ms (let's say 10.5) is pretty close to 8.5-9.5 (let's say 9ms). It's within reach, at the least! Instead of 36% speed, we're now running at almost 86%!

Further optimizations are certainly possible, and I'm excited to see where it will take me.

### unchecked()
unchecked() is not ideal, though. If I want to build it with bounds-checking on, I have to go through and remove all of that. NOT good for actually developing code. But the community is great. I just opened an Issue: https://github.com/AssemblyScript/assemblyscript/issues/2536

With the responses I've gotten so far from the community, I expect it'll be a thing pretty soon. Maybe I'll even make a PR for it! 

### The path forward
So, AssemblyScript is a bit of a step back for JSMoo, because it requires an extra step of compiling the code first. This defeats one of my goals of making it so you don't need a local development environment to tinker with JSMoo.

HOWEVER! AssemblyScript's compiler can be run as a WebAssembly package inside your browser! I plan to add on-load compilation of the AssemblyScript source at some point in the future, and I can preserve that ability to do it all from your browser!

Due to its similarity to JavaScript, I think the code is pretty readable and modifiable for anyone who knows JavaScript, so my goal of keeping the code understandable to as many people as possible also seems to be preserved.

This may also be the way forward for Just-In-Time dynamic re-compilation for more complex emulators. Time will tell.