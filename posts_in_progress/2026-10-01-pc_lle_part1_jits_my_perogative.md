## JIT's My Perogative

Although I've done some Dreamcast work since I'm A JIT 4 W, I'm mostly focused on low-level IBM PC emulation now. 

To my surprise, a LOT of what worked well for a Dreamcast JIT has *not* worked well for a x86-32 JIT.

I've modified MIR extensively, and made a lot of different architectural changes. I've wanted to write about what and why for a while now.

### Preamble

#### Platform differences
The SH4 in the Dreamcast is a 200MHz RISC chip, with fixed 16-bit instructions, 1kb, 4kb, and up minimum page sizes on the MMU, and it mostly gets a bit under 1 instruction per cycle. Despite the processor being released in the same year as the Pentium II and containing FPU SIMD, it's also a low-power/embedded device with small cache and not as deeply pipelined as they could have for the time. It's a great part, but it doesn't match up to what Intel was doing at the time. (Of course it's a lot cheaper than Intel's chips!)

Around the same time, Intel’s Pentium II launched at up to 300MHz, with variable-length x86 instructions and a 4KB minimum MMU page size. Its more deeply pipelined, out-of-order core could retire up to three micro-ops per cycle, backed by 16KB each of instruction and data cache and 512KB of L2. It also had integer SIMD through MMX. It was a much more aggressive desktop design, carrying decades of x86 compatibility—and all the decoding, flags, segmentation, and execution-mode complexity that an emulator inherits if it wants to do low-level emulation.

#### LLE vs HLE for IBM PCs...and XBox?
That complexity COSTS.

If you want to run modern games, or even games from like 2010, you generally will use an HLE such as GameHub, Winlator, or GameNative. These basically take a x86-64 binary, and JIT it into local OS process space, and use a variant of Wine and other compatability layers to get games to run. Because of this they can do a lot of native mem mapping, use a simplified memory layout, and make many assumptions an LLE emulator can't. 

If you're doing it all LLE, you are at the OS you're emulating's whims. You have to implement things a lot more strictly, and include support for a lot of crap that HLE just doesn't need to worry about.

I set my original goal at Deus Ex (2000), which has a minimum specs of Pentium II at 400MHz. I soon moved to considering LLE XBox emulation, which runs a Pentium III at 733MHz.

I'm also developing this on an Apple Macbook Air M4 first, so it's aarch64-specific. However, one wrinkle is the way Apple restricts page boundaries to 16kb on MacOS. This would cause me no small amount of grief.

#### Software differences
Perhaps most importantly, 95+% of Dreamcast games don't touch the MMU. Every single time you boot Windows, you are using the MMU. That adds SIGNIFICANT overhead, to every jump and memory read and write. It adds not just address translation for reads and writes, but also for block linking/jumps, as well as faults on unmapped or improperly privileged accesses. 

I did get my Dreamcast MMU fairly fast, but it was easy-mode compared to x86 in many respects.

### JIT architecture
From my SH4 JIT, I've taken a few pieces. Chief among them, the CI/JIT interchangeability.

MIR is great. Its optimizations can make code go 30-60% faster than unoptimized straight translation. However, these optimizations take TIME. Compilation can be quite slow during Windows boot especially, or level loading, etc.

To ameliorate this issue on SH4, I put effort into the CI (Cached Interpreter). I did basic profile-guided optimization, that only marked a block for compile after it was executed 4 times. I also added asynchronous compile on worker threads, so that this compile wouldn't block CI execution. This way you get the CI as the minimum experience, which isn't BAD, and the JIT as the warmed-up execution experience.

I really liked this dual-tier strategy, so I've used it in the PC-LLE project. 

MMU was much more difficult to optmize for PC though, and I had to take a very different approach. 

For instance, WindowsCE used ASIDs to distinguish different processes with different mappings, a hardware identifier. This led me to be able to compile blocks and pass in an "aperture," maintaining up to 16 different "mappings," only one or two of which saw most use. Windows 98 and x86 does NOT have this ability, and keeping these apertures by determining some kind of identity that Win98 doesn't share with the hardware, has been difficult.

The SH4 tends to have less memory accesses due to its greater number of GPRs and simple memory addressing modes.

There are other reasons but suffice it to say that SH4 is conducive to fast MMU emulation in ways that x86-32 is not. Because of all this, I did a lot less work on certain parts of SH4.

Although a naive PC-LLE CI could go VROOM in 386 DOS games, switching to win95+ paged Protected Mode slowed things down a LOT.

#### General Rules
As a general rule, you want your JIT to spend as long as it can executing recompiled code, with as few call-outs or returns to C code as possible, because that causes "register spills" (where state has to be saved to and reloaded from the stack). You also want and as little bookeeping and overhead as possible, since that gets multiplied by millions and slows down execution.

You also need to be able to detect Self-Modifying Code (SMC), which is used fairly extensively in parts of x86 code, as well as detect when code blocks are overwritten in guest RAM. Especially due to the way Windows re-uses pages, this can be an issue.

And then there's permissions, segmentation, Virtual 8086 mode, protection rings, etc. Windows 98SE makes extensive use of real-mode (286-style) drivers still, and the OS protects itself not with ASIDs like on SH4, but with segments. A context switch means a whole new mapping which is expensive.

### Memory access types a JIT cares about

There are two common cases for memory access that are related but separate:

1) Code wants to read or write memory that may or may be MMIO, RAM, etc.
2) A block wants to jump to another block, so has to be able to look up the address for the corresponding block quickly (SH4 was largely amenable to hard-links where these are baked in, even in MMU modes

### Typical HW fastmem

In order to do the first one, I've done a fairly typical "hardware fastmem." Reads and writes are done as the guest address plus the offset of a mapped area of memory. RAM reads and writes go directly through (though with some version tracking info we'll get to), and MMIO causes faults. The MMIO causes the cell of the instruction that caused it to be marked as "unsafe," so the block is re-compiled with slow-path RAM access for that specific instruction. Pages that contain code also get write-protected, and a write to them causes a specialized routine that invalidates nearby code pages. 

The only wrinkle is those 16KB minimum pages for Apple. Because of that, for each x86-32 4kb page, I have to reserve an Apple 16kB page. I have to put in if's for if an access may straddle pages and split it up, as well as do some arithmetic to expand from the 4kb to 16kb page size for any address. Because of this, I have a long-term plan: an EL1 hypervisor that allows me to run with 4kb pages like Rosetta. I prototyped a large amount of this but haven't completed it yet.

### Block linking

The second area is the really interesting (to me) part. When one block is done and needs to head to the next, there are a million schemes I've tried, but The Scheme is the best.

A block knows its next desination address in virtual memory, not physical memory. Its destination may have had its page changed, or been overwritten, or any number of things, so you can't just store the address of the next block to execute without guards. An "edge cache" can help, which says "if this is the same virtual address as last time, use the same physical address->block identity," but this didn't take me very far on x86-32.

Instead, just like how typical HW fastmem allows us to use the MMU to speed up reads and writes, The Scheme allows us to use the MMU to speed up block linking.

Two separate 32GB address spaces are created, the SLOW and the HOT ones. 32GB because that's 8 * 4GB (the 32-bit addressing size): 8 bytes per 1 valid destination byte in the memory space.

The point of the scheme is to reduce inter-block links to a very simple operation.

JIT: do addr << 3, add to aperture base, load that. This is P. Jump straight to P to continue execution. 

CI: do addr << 3, add to aperture base, load that. This is P. Now, subtract 8 from P, load THAT value, and tail-call that. 

The scheme works this way:
Any uncompiled/untouched address points to a "compile this with CI" helper. P will be a thunk to C and tail-call of the "compile this to CI" function, while P-8 will be a pointer to that tail-callable "compile this to CI" function. We only need one "block" like this, and any unmapped areas can point straight to it.

A address that has had CI compile but not yet JIT yet, P will be the start of a thunk that will tail-call the CI, and the CI will tail-call into the next CI block directly with P-8.

For an address that has had a JIT block compiled and installed, P will be that compiled code, where P-8 will point to a shared thunk from CI->JIT that will enter JIT mode and call P.

For a block that has been invalidated, P points at a hash-based verifier, while P-8 points to a tail-callable C version of the same.

This way, there are *no edges* except for not-yet-compiled ones (and SLOW path ones, we'll get to that) that can cause a return to C, or take any time at all to look up and execute. We map pages in lazily to this aperture to cut down on MMU traffic (the same as with HW fastmem). There's no multi-level cache or slow loojup. There are no guards, or if's. JIT and CI both just load an address (CI has an additional indirect load) and go.

### Of course it's not quite THAT simple
After all, you want as little extra cruft in the form of bookeeping and overhead as possible. There are several things that can cause a block to fail out to the SLOW path.

I keep a global state, which basically has "HOT is available," as well as FPU disabled, and some other key bits. At each block entry, a compare is run against that global state with a baked-in constant. If the "key" doesn't fit, the block aborts out to the SLOW Path. This happens literally only a few times per frame (during game execution at least), so that path can do all the expensive translations etc. that the HOT path doesn't.

I also recover from faults by using a "reconstruction map." This is one of a few changes I've made to Mir, which basically added metadata that says "this PC has these registers in these spots." It allows a fault handler to commit processor state that was "in flight" during the fault. All instructions are designed to fault before mutating state, so like an RMW increment using EAX as address would look like

```
READ (EAX) -> temp (can fault)
temp++
WRITE temp back to memory (can fault)
; faults are done, we can mutate processor state
EAX = temp
flags = results of this
```

I also modified MIR to "see" the ARM PSTATE flags and only update PSTATE when needed. This allows native flag compares as well as minimizes PSTATE traffic which tends to be fairly slow.

### The FPU disable bit: a SLOW-path success story
Windows (on both SH4 and IBM PC) makes use of the FPU disable bit. What it does is, on thread switch, it just disables the FPU. The first FPU operation will then cause an exception, and if it's in a different thread from last time, only THEN does windows do an FPU context switch. This allows integer-only threads to pay no FPU cost, and speeds things up.

Of course, we don't have that bit on AArch64.

So, "FPU enabled" is checked as part of the block-entry guard if a block contains FPU instructions. If so, it is interpreted until an FPU instruction is reached, which checks the bit and throws the guest exception.

This only happens a few times per frame, like all SLOW path executions.

### The SLOW path
The SLOW path does segment and permissions checks, as 

### Lean and mean but not fast yet
So, block transitions are extremely lean. Block guards are a single load, mask, conditional jump on fail. CI->JIT and JIT->CI as well as determining when to compile a block are about as fast as possible without direct links, and are unconditional. The same code mapped to multiple locations is only compiled once. 

Despite all these and more, I'm only emulating at about 300-500MHz with Pentium II timings (including rough pipeline timings) in sustained gameplay on my 4.4GHz Apple M4.

I have a plan to make an Elevation 1 hypervisor which allows 4kb page mapping to cut 5-8 instructions per access (check straddle, change 4->16kb page stride, do access) down to 1 instruction per access. But this doesn't really fully explain why my throughput is so low.



