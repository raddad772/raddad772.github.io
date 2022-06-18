## Emulating in Javascript, and also, emulating the 65c816 with cycle accuracy and speed

And why they're both a "fun" challenge
---

### Cycle-accurate SNES emulation in JavaScript? Are you LITERALLY CRIMINALLY INSANE?
I can't speak to my sanity, but for some reason I decided to do as much of the development as I can directly on my iPad. That seriously limits my options.

But I think it's possible. I've seen a few failed projects, but I'm a senior software engineer. I wrote a pretty decent NES emulator 20 years ago. I love low-level projects and embedded programming. I'm not a fan of JavaScript, but that just adds to the fun, right?

Right?

### What's the big deal about SNES?
To emulate the SNES with great accuracy, it's actually roughly right below the level of the Sega Saturn. This is one reason we don't have many SNES emulators, but have literally hundreds of SMS, NES, and GameBoy emulators.

### The fun of JavaScript
So it turns out, if you want to have fast JavaScript, you're in for a bumpy ride. It's important to optimize your code not for an underlying architecture, but for the Just-in-Time compilers that will compile your code, and the VMs it will then run on.

Since I'm not a super JavaScript fan, I'm leaning on the experience of others to some extent. Big switch statements are right out, because it'll cause JIT compilers to give up and interpret. Same with changing the "shape" of your data, like adding members outside of the constructor.

This is one of many reasons I chose an approach that involves computer-generated code. I can (theoretically) make changes in a few places, re-generate the source that actually runs the CPU, and measure performance differences.

### Addressing pain
The 65c816 is kind of a pain. Yes, I know MODR/M gives people nightmares, but the 256 available opcodes are constructed from two types of primitives: roughly 90 instructions and roughly 25 unique addressing modes. That number extends to 48 when you take in variants such as Read-Modify-Write.

An example of this is AND, a simple bitwise operation. Simplified JavaScript for it might look like this:

```
function AND() {
    A = A & operand;   // Perform operation
    P.N = (A & 0x8000) >> 15; // Negative flag
    P.Z = +(A === 0);  // Zero flag
}
```

This single instruction is available in a ton of different addressing modes - 15, to be precise. An addressing mode basically refers to how the processor figures out exactly which data to do the operation to. Each different mode accesses data in a different way.

We don't want to write 15 (actually 60, but I'll get to that) versions of the same function, so we want some kind of code that will generically access the data from memory, then call the actual AND algorithm, then complete the instruction. We could then theoretically apply this to other similar instructions like ADC, ORA, and LDA, and save ourselves a lot of code.

In a simplified context, an Absolute (which is directly addressing memory with 16 bits) addressing function might look like this:

```
function Absolute(instruction) {
    let operand = mem_fetch(address);
    if (instruction === AND) AND();  
}
```

Except it's not nearly so simple. Different instructions use the same addressing mode but behave very differently. For instance, AND $1000 and INC $1000 both function differently, despite both using absolute addressing.


```
function INC(operand) {
    operand++;
}


function Absolute(instruction) {
    let operand = mem_fetch(address);
    if (instruction === AND) AND();
    if (instruction === INC) {
        INC();
        mem_write(address, operand);
    }
}
```

So far, not so bad. But what about when INC is instead called on the Accumulator, or X, or Y? Now our INC function is looking more like this


```
function INC(operand, mode) {
    if (mode === ACCUMULATOR) {
        A++;
        P.Z = +(A === 0);
        P.N = (A & 0x8000) >> 15;
    }
    else if (mode === X) {
        X++;
        P.Z = +(X === 0);
        P.N = (X & 0x8000) >> 15;
    }
    else if (mode === Y)
        ...
    else if (mode === mem)
        operand++;
        ... 
}

```

You can see this is getting laborious.

There’s other permutations I’m not covering explicitly too, like the difference between load absolute and store absolute. 

If you know anything about modern processors, you will know that running quickly on them depends heavily on speculative execution. Their super long pipelines slow to a crawl if they miss the prediction of an if statement. If statements are one of your greatest enemies, and that does prove true even when speaking about interpreted or JIT virtualized languages like Python or JavaScript.

Instead of using if’s, you could write a bunch of different versions of INC:

```
function INCA() {
    A++;
    P.Z = +(A === 0);
    P.N = (A & 0x8000) >> 15;
}

function INCX() {
    X++;
    P.Z = +(X === 0);
    P.N = (X & 0x8000) >> 15;
}

// and the same for INCY and INCMEM and anything else applicable

```

This approach has a lot of downsides. What if there's a bug in the repeated code testing for the Negative flag? What if fixing that and copy-pasting it 20 times adds another bug? 

Of course, function calls themselves are a bit of a slowdown, due to the way JIT compilers trace hot code paths. Ideally, you'd create a separate function for every possible combination of addressing mode, instruction, and status bit.

### The status bits

There's one more BIG complication here, first! You see, depending on the status of 3 different processor flags, individual instructions AND addressing modes can behave differently.

The X flag toggles the Index (X & Y) registers between 8 and 16 bits. The M flag toggles memory accesses and the Accumulator between 8 and 16 bits. Also, the processor can be in Emulation mode (E), or not. These 3 bit flags (theoretically) make 8 times as many opcodes to emulate. In reality the number is 5, because when E is 1, M and X are also forced to 1.

So an INC in 8-bit mode on a piece of memory will have at least 2 less cycles than an INC in 16-bit mode, and will also need different algorithms. You'll either need a lot of IF statements, some VERY big switch statments, or a lot of nearly-identical code like Absolute8bitRead, Absolute16bitRead, Absolute8bitReadModifyWrite, Absolute16bitReadModifyWrite, EmulatedReadModifyWrite, EmulatedRead, and 8- and 16-bit versions of all those instructions.

That's just for one instruction and one addressing mode, and doesn't even cover 3 more sub-types of the Absolute mode.

That's a ridiculous amount to hand-code!

### Code generation to the rescue

So, you are either going to have a lot of IF's in your code, which will slow it down a LOT, or you are going to write code to generate the functions that end up executing. This is the approach Higan and JSMoo (my SNES emulator) both take, although we go about it in different ways.

Basically, instead of looking up which function to execute based only on the opcode in a 256-entry table, you end up with a 1280-entry table that takes into account the current state of the E, M, and X bits - or some equivalent. So now you're basically emulating 1,280 opcodes instead of 256.

Higan's approach could be the subject of a whole blog post, but this blog is about JSMoo. Basically, the code uses the C preprocessor to re-include the same opcode table over and over, and uses macros to redefine certain behaviors.

JavaScript doesn't have a preprocessor, and I don't want to use coroutines like Higan does. So I take the approach of generating the code from a different piece of code.

This does have some downsides. The generated file is currently multiple megabytes, and minifies to around 800k. That's not too terrible if it actually runs fast.

However, it has many upsides, too. It allows me to build instructions out of primitives, debug multiple opcodes at once, and be as assured as I can be of correct behavior.

The final generated code looks like this. Keep in mind another complication we haven't discussed is how to emulate the bus states and individual cycles, which is the reason for the switch statement. And this is just for INC absolute with E=0 M=0 X=0. There are actually 4 different versions of this specific function, and here's just one, with added a few comments for clarity.

```
function(regs, pins) { // INC a
    switch(regs.TCU) { // TCU is Timing Control Unit, a real register on a real 65c816 used for this same purpose. It increments by one each cycle.
            // INC a E=0 M=0 X=0
            //case AM.Ad
        case 1: // 2 - where we start the fetch of the first byte of the address of the operand
            pins.Addr = regs.PC; pins.BA = regs.PBR;
            regs.PC = (regs.PC + 1) & 0xFFFF;
            break;
        case 2: // 3 - where we start the fetch the second byte of the address of the operand
            regs.TA = pins.D;
            pins.Addr = regs.PC; pins.BA = regs.PBR;
            regs.PC = (regs.PC + 1) & 0xFFFF;
            break;
        case 3: // 4 - where we start fetching the first byte of the operand
            regs.TA += pins.D << 8;
            pins.Addr = (regs.TA); pins.BA = (regs.DBR);
            break;
        case 4: // fetch_rmw_8or16 16L - where we start the fetch of the second operand byte
            regs.TR = pins.D;
            pins.Addr = (pins.Addr + 1) & 0xFFFF;
            break;
        case 5: // fetch_rmw_8or16 16H - we have the second operand byte, time to perform the actual instruction
            regs.TR += pins.D << 8;
            pins.PDV = 0;
            // instruction code follows, taken from INC16()
            regs.TR = (regs.TR + 1) & 0xFFFF;
            regs.P.Z = +(regs.TR === 0);
            regs.P.N = (regs.TR & 0x8000) >>> 15;
            // instruction code ends
            break;
        case 6: // finish_rmw mem16 H - where we begin writing the result back to memory
            pins.RW = 1; pins.PDV = 1;
            pins.D = (regs.TR >>> 8) & 0xFF;
            break;
        case 7: // finish_rmw mem16 L - where we finish writing to memory
            pins.Addr = (pins.Addr - 1) & 0xFFFF;
            pins.D = regs.TR & 0xFF;
            // Following is auto-generated code for instruction finish
            break;
        case 8: // cleanup - where we increment the PC to set up the next opcode fetch. technically cycle 0 of the next instruction
            pins.Addr = regs.PC; pins.BA = regs.PBR;
            regs.PC = (regs.PC + 1) & 0xFFFF;
            pins.RW = 0;
            regs.TCU = 0;
            break;
    }
},
```

Don't worry about all that stuff about pins. For some reason, I decided I wanted to make my CPU interface through pins, like a real one. It actually ends up being convenient for the purposes of the SNES, which I'll probably discuss later.

There are over a thousand other functions that look just like this. The generated file is over 50,000 lines long. The code to generate it totals an order of magnitude less. And if I want to change the overall approach for purposes of optimization, I only have a few places to do it. Lots and lots of win.

A specific example of that win is in debugging. If I find and fix a bug in the addressing code for LDA a, I will also be fixing that same bug in a few other instructions at the same time, for free. 

One interesting thing of note is that by far, the majority of this code by volume is not actually dealing with an instruction. In fact, most instructions on the 65c816 are so simple they fit in just a few lines - it's all the other things the processor is doing that are a pain.

### Wrapping it up

So that's some of the fun involved in emulating the 65c816, and doing it in JavaScript. So far, this approach is fruitful, and it looks reasonably fast despite its cycle-accuracy. Only time will tell about how well it performs as a component of a more complex system.

Next up: adventures in testing a CPU core for an unpopular CPU!