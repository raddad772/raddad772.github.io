## I'm a JIT 4 W(indows)

All those percentages in Part 2 came off an Apple M4, a machine that demolishes indirect branches and dependent loads. It was time I got serious about a slower x86 PC: a Ryzen 5 4600H laptop (Zen 2, six cores, 45 watts on a good day) with a GTX 1650, running Windows. And later, a 15W version of the same core with a slower iGPU.

This post covers the three weeks it took to make the JIT fast on that machine.

### The port

Getting a working port only took about a day. MIR needed a couple of Win64 fixes in my fork after I got the dev environment set up.

However, the emulator that was running over 1400 MIPS on M4 was running roughly 200-300 to start with on x86-64.

Technically, only 200 MIPS is NECESSARY, though it won't be very smooth, and good luck on a weaker processor. The higher we crank the MIPS, the better the emulator will run on any given PC.

I defaulted to the old, slower software fastmem, which used range checks before doing RAM read/write or selecting an MMIO call.

Since Part 2, Aarch64's memory fast path has used typical hardware fastmem: guest RAM lives in a shared section mapped 28 times over a 512MB reservation - one view per cached/uncached mirror. So (guest_address + base_register) = host_addr, and anything that isn't RAM faults into a handler that patches that code block and calls the slow path. All of that machinery was POSIX: `mmap(MAP_FIXED)` over a reservation, `sigaction(SIGSEGV)` to install handlers.

Windows has equivalents for every piece:

```c++
//   mmap(PROT_NONE) reservation  -> VirtualAlloc2 (MEM_RESERVE_PLACEHOLDER)
//   mmap(MAP_FIXED|MAP_SHARED)   -> placeholder carve + MapViewOfFile3(MEM_REPLACE_PLACEHOLDER)
//   munmap (keep the hole)       -> UnmapViewOfFileEx(MEM_PRESERVE_PLACEHOLDER)
//   sigaction(SIGSEGV/SIGBUS)    -> AddVectoredExceptionHandler
//   ucontext register access     -> the CONTEXT record in the exception pointers
```

This gave +38% SH4 JIT throughput over the software path. However, 38% faster than slow is still slow.

### Windows charges by the syscall

Some things that POSIX does cheaply or lazily, Windows does expensively or eagerly.

**Find #1: demand-paging** Those 28 RAM views are mapped lazily. The first touch of each 4KB page faults it in. On the Mac that didn't even show up on a profiler. On Windows, the first time a game streams a level through its uncached mirror, you get 100,000–300,000 faults per second. This meant ~40% of wall time spent in the kernel, and a huge frame hitch. The fix was fairly simple:

```c++
// setup(): read-touch every page of every alias ONCE, so demand paging
// is paid at init instead of mid-gameplay
for (auto *view : ram_views)
    for (u32 off = 0; off < RAM_SIZE; off += 4096)
        (void)*(volatile u8 *)(view + off);
```

**Find #2: excessive recompiles.** Half-Life would run fine, then crater for a second, then run fine. The JIT's code space has a size cap, and my strategy for dealing with blocks is just "drop them all and start over" if it fills. The counters I put in said that it was firing every ~110 frames, which made no sense. On Windows, MIR allocated things differently from Mac. The best fix for this was teaching MIR (in the fork) to carve all code out of one contiguous reservation. Flushes per run went from 12 to 1 (I perform a flush on MMU enable), and worst frame time went from 1201ms to 237ms.

The tram's average went way up by troubleshooting fairly basic structural differences between the platforms like this. However, being an MMU game, Half-Life is severely handicapped regardless in performance at this point.

### ~570 MIPS: AICA, Again. (Part one of again.)

The flame graph's next biggest non-JIT tower was AICA audio — and its ARM7 was still running a plain interpreter, re-decoding every instruction, every execution. On M4 this took like 8% of CPU time even at billions of instructions per second, but on Windows took up more time than you'd believe. I'd already written a basic cached interpreter for the ARM7DI for the GBA and NDS, it just was never wired up for AICA. Wiring that up took Soul Calibur from 447 to 570 host MIPS.

### ~776 MIPS: the Part 2 optimizations, ported

Next, I brought over a bunch of optimizations from Part 2. Hard block links, baked links, devirtualization, plus a pile of x86-64-specific emission work. Two highlights:

- **Materialize the fastmem base once per block.** Every access site was loading `cpu->fastmem_base` and adding, to rematerialize a value that's constant for the life of the native block. I hoisted it into a register-allocated var at block entry and every site reduced to one ADD. +2.4%, SC 738 → 756.

- **Const-address folding.** Per-block constant propagation over guest registers. If I see a `MOV #imm`, MOVA, or currently-bbaked literals, I track it through copies, adds, and shifts. A data access whose address is provably a RAM constant compiles to a single displacement access. In SC's fight loop, 53.7% of data accesses have const-RAM addresses and another 45% are the literals themselves. SC 756 → 776.

### Stretch existing MIPS a bit further

Since Part 1, "MIPS" has meant 1 instruction = 1 cycle. The real SH4 doesn't get 1 instruction = 1 cycle, however, though well-tuned code can come close. It's dual-issue: two instructions retire per cycle when they pair, and the Software Manual spells out exactly when they pair. It has instruction groups, co-issue restrictions, per-instruction issue rates. Memory accesses also take at least 2 cycles. So I implemented it so that the emulator now charges blocks from a per-opcode attribute table, built by a script from the documentation's table. A pairing walk happens at block-compile time, a small memory-pressure surcharge stands in for cache/SDRAM latency, and tada: a roughly correct cost model.

I took a bit of inspiration from Flycast as well, and penalized certain MMIO accesses and short loops. This charges more for idle-loops that are waiting on something, while keeping overall instruction throughput the same; this rests behind a core option. 

Games now run at the "right speed," and we get more instructions-per-MHz. Instructions per second stopped being the best metric, so from here on I use eq-MHz, or equivalent MHz.

Every performance number in this post so far, and every number in Parts 1 and 2, is in the old unit. The two units happen to be within ~10% of each other on SC (dual-issue pairing and the memory surcharge nearly cancel there), so the upward trajectory is roughly the same, but our benchmark numbers are even less equivalent now.

(For the curious, some games get closer to 25%-35% difference, Soul Calibur is generally well-tuned).

### The economics reverse

Part 2 closed saying that per-edge optimizations were as good as they could get. That only turned out to be true for the M4. Zen 2 with the Win64 ABI is a different beast, and three of Part 2's conclusions flipped outright.

**Register residency** In Part 2, step 11, I pinned guest R0–R6 into host callee-saved registers, and got +3.4% on SC. After I ported it to x86-64, I A/B'd it: pin 708 vs no-pin 730. aarch64 gives MIR 31 GPRs to allocate and deal with, including many callee-saved registers that are basically free-use temporaries. x86-64 gives it 11 after the JIT's own infrastructure registers, and taking more away costs the allocator more in spills than guest-register residency saves.

**FRCHG call vs inline** The SH4's FRCHG instruction swaps the two 64-byte FP register banks. On the M4, inlining the swap into the block beat calling a helper. But MIR had no vector IR, so the inline swap became 32 scalar 8-byte moves mid-block, and on Win64's tighter register file all that inline bulk has a price. The C helper, meanwhile, is three memcpys that clang vectorizes into ~16 unrolled `movups`:

```c++
void REGS::FPSCR_bankswitch()
{
    memcpy(&fb[2], &fb[0], 64);   // clang turns this whole thing into
    memcpy(&fb[0], &fb[1], 64);   // a straight run of movups
    memcpy(&fb[1], &fb[2], 64);
}
```

The call won with a 3.7% difference. It was the same story for FTRV, the 4x4 matrix-vector transform: the inline version is ~52 scalar FP ops; a helper written so the SLP vectorizer can see the contiguous loads compiles to 4 `mulps` + 3 `addps`, and wins +4%. "Inline the hot thing" is not ALWAYS a win (though, a more-optimized JIT may have found a win there).

### The light-call saga

The FRCHG result bugged me, though. Fine, the trampoline's body is better than the inline, but the call site still pays the full ABI cost: every caller-saved register the block has used gets spilled around the call, for a helper that touches four xmm registers and nothing else. What if the JIT could know that and issue slimmer calls to specialized functions?

(Note: in my later PC-LLE days, I've gotten more comfortable modifying Mir and adding vector instructions. I was not at this point in SH4 dev yet though.)

So I added a "light-call ABI" to the MIR fork. A prototype flagged light promises to clobber only `{rax, xmm0–5}`, and the register allocator keeps everything else live across the call, eliminating spills. The light-called function will spill most things it wants to use. And the helpers became naked functions:

```c++
__attribute__((naked)) static float jit_tramp_sqrt_light(float) {
    __asm__("sqrtss %xmm0, %xmm0\n\tret");
}

// FRCHG: flip FPSCR.FR (bit 5 of byte FPSCR+2) + swap the 64-byte banks.
__attribute__((naked)) static void jit_tramp_frchg_light(void *) {
    __asm__("xorb $0x20, 0x8e(%rcx)\n\t"
            "movups 0x98(%rcx), %xmm0\n\t" "movups 0xd8(%rcx), %xmm1\n\t"
            "movups %xmm1, 0x98(%rcx)\n\t" "movups %xmm0, 0xd8(%rcx)\n\t"
            /* ...three more quads... */
            "ret");
}

static_assert(offsetof(core, regs.FPSCR) + 2 == 0x8e, "layout drifted; fix the asm");
```

sqrt, FRCHG, FTRV all got the treatment. All three validated as correct. It's some of the tightest code in the whole emulator. The 4x4 transform costs its ~19 SSE ops and nothing else. Looks good and fast, right?

Nope.

The A/B benchmarks never cleared the measurement noise floor. Beautiful, correct, unmeasurable code with hardcoded structure offsets and a paired prototype-flag/helper contract, could only be worth maintaining for a measurable gain.

### AICA, part two of again: JIT number two

Profiling brought me next back to the ARM7. Even after the cached interpreter was wired up, the ARM7 was still around 10% of all host time on this platform. So time to make a (basic, not super-tuned) ARM JIT! The throughput is, after all, only a 2-6MHz.

I wrote an ARM32 to MIR translator. I architected it with a simple, flat entry table of u32 offsets-from-code-base instead of full pointers, which would be double the size. Trying to be kind to Zen 2's cache.

This was worth +5.5% on SC over the ARM7 cached interpreter. This sounds modest until you remember that the ARM7 was maybe a tenth of host time — within its budget, that's most of the meat.

### JIT number three

The AICA doesn't just have an ARM7. It also has a DSP: a 128-step microprogram that runs every sample, 44,100 times a second, doing effects mixing. That's more throughput than the ARM7 has. My initial naive interpreter walked all 128 steps re-evaluating ~10 field-select branches per step — BSEL, XSEL, YSEL, ZERO, SHFT, on and on — on fields that are constant for the life of the program. Zen 2's branch predictor falls down hard on this.

So I compiled the microprogram. I specialized every step on its constant fields and emitted only the taken paths:

```c++
// interpreter, per step, per sample, 44100 Hz:
if (op.XSEL) x = temp_val; else x = mems_val;
if (op.YSEL == 0) y = frc_reg; else if (op.YSEL == 1) y = coef[step]; ...
if (op.ZERO) acc = 0; else if (op.BSEL) ...   // ~10 of these

// JIT, per step: the op fields are constants, so emit ONLY the taken path.
// A step with XSEL=1, YSEL=1, ZERO=0 compiles to, in effect:
x = temp[(step + mdec) & 0x7f];
acc += ((i64)x * coef_s[step]) >> 12;
```

I also only bother compiling DSP at frame start, and if the program is modified it gets interpreted for just that frame, because why add a bunch of compile overhead during an update? This got a huge +8.2% on SC.

Three JITs in one emulator now.

### Attack of the Undefined Behavior

All of the above measurements are done with A/B runs, and I'd had a rather large noise amount between runs that made me suspicious for a while now. At times I'd do 3-repeat ABBA runs and get weird results. So, I built determinism checking into the bench harness ny running the workload twice, and checking the guest cycle count, the ARM7's retired-instruction count, and a final-frame screenshot hash. If they didn't all verify, it automatically failed.

And...the runs did NOT match. The ARM7's instruction count swung between 109 million and 440 million per run - a 4x difference, across identical runs of the same recorded workload, while the picture stayed pixel-identical.

The ARM7DI's multiply timing models the Booth algorithm early-out: cost depends on the number of leading zeros in the multiplier.

```c++
// before:
u32 n = __builtin_clz(multiplier) >> 1;
if (n > 16) n = 16;                       // safety clamp, right? right?
idle(n);
```

Little did I know, `__builtin_clz(0)` is undefined behavior, and a multiply by zero is not rare. At runtime, `bsr` with a zero input leaves the destination register unchanged, so the result comes from whatever garbage happened to be in that register. And compiler optimizations made it worse. The optimizer is entitled to assume UB can't happen, and as such, it assumes the input is nonzero. It then proves `n >= 0 && n <= 15` since it's 1-31 >> 1. It then deletes the guard I wrote. One multiply-by-zero then charged a wildly random idle - I measrued and sometimes it was up to 1.4 billion cycles. The ARM7's interrupt timing lurches, and the workload is different in every process while the game plays on unbothered.

The plain interpreter was clean because the compiler happened to keep the clamp. So the bug looked like a cached-interpreter timing divergence, and I hunted it for way too many hours.

```c++
// after: std::countl_zero is DEFINED for 0 (returns 32) --
// countl_zero(0) >> 1 == 16, the intended max cost.
u32 n = std::countl_zero(multiplier) >> 1;
```

I of course audited every functional `__builtin_clz` in the codebase and changed them to `std::countl_zero`, which removed the entire bug class. This flushed out another bug in the PS1 GTE as well.

After this, A/B's got a lot more stable, with closer to 2-3% thermal variance per run, but I haven't revisited some earlier conclusions.

### Where it stands

Call it roughly 4x realtime for the 200MHz guest, on a six-core laptop from 2020, at the wall wattage of a bright lightbulb. The M4 still reads ~2x higher on the same fight, but this seems good enough. 

I did further development on a 15W version of the same chip with an iGPU, and I got Half-Life up to like 560 MIPS in the tram benchmark with further optimizations, but really I'm getting kinda tired of this blog post series. I've been working on Intel CPU JITs and want to start writing about those, because SH4 feels like easy-mode in comparison.

### Two last interesting notes for Dreamcast/JIT emu-devs

There are two final remarks from later optimizations to make here.

1) I saw that the SH4 supported 1kb minimum page size, and that some were instantiated, so assumed it'd always be that way. This meant a lot slower paging even in Windows which supports 4kb pages.

Well, I eventually measured, and no WindowsCE games, anywhere that I got in gameplay, have any 1kb pages allocated outside of initial booting process. So I slow-pathed 1kb accesses, and treated it AS IF there was 4kb minimum page size, and got a fairly significant speed-up in games.

2) On the M4, 64-bit pointers to everything are everywhere. I tried shrinking them to 32-bits (which means it's now base + offset instead of a direct pointer) and it shrunk code size but was a wash overall. On Ryzen, shrinking pointers to 32 bits was often a big win, and I got 5-8% speed increases in games doing this in a few different places.