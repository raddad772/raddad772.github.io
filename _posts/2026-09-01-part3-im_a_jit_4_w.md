## I'm a JIT 4 W(indows)

Part 2 ended with a caveat: all those percentages came off an Apple M4, a machine that eats indirect branches and dependent loads for breakfast. So for Part 3 I got serious about a slower x86 PC: a Ryzen 5 4600H laptop (Zen 2, six cores, 45 watts on a good day) with a GTX 1650. Running Windows.

This post covers the three weeks it took to make the JIT fast on that machine. Getting a working port only took about a day. The next part mostly covers the "fast" part. 

### The port

The build was the easy part: llvm-mingw (clang 22 + libc++), same compiler family as everywhere else, so the `musttail`/`preserve_none` machinery from Part 1 just works. MIR needed a couple of Win64 fixes in my fork and came along too.

The interesting part is fastmem. Since Part 2 the JIT's memory fast path has used *hardware* fastmem: guest RAM lives in a shared section mapped 28 times over a 512MB reservation - one view per cached/uncached mirror - so a guest address plus one base register IS the host address, and anything that isn't RAM faults into a handler that patches that code block and calls the slow path. All of that machinery is POSIX: `mmap(MAP_FIXED)` over a reservation, `sigaction(SIGSEGV)` to install handlers.

Windows has equivalents for every piece:

```c++
// the actual translation table, from dc_fastmem.cpp:
//   mmap(PROT_NONE) reservation  -> VirtualAlloc2 (MEM_RESERVE_PLACEHOLDER)
//   mmap(MAP_FIXED|MAP_SHARED)   -> placeholder carve + MapViewOfFile3(MEM_REPLACE_PLACEHOLDER)
//   munmap (keep the hole)       -> UnmapViewOfFileEx(MEM_PRESERVE_PLACEHOLDER)
//   sigaction(SIGSEGV/SIGBUS)    -> AddVectoredExceptionHandler
//   ucontext register access     -> the CONTEXT record in the exception pointers
```

Hardware fastmem on Windows: +38% SH4 JIT throughput over the software path, in one commit. However, 38% faster than slow is still slow.

### Windows charges by the syscall

The next two finds are the same lesson: some things that POSIX does cheaply or lazily, Windows does expensively or eagerly.

**Find #1: demand-paging** Those 28 RAM views are mapped lazily — the first touch of each 4KB page faults it in. On the Mac that was never visible. On Windows, the first time a game streams a level through its uncached mirror, you get 100,000–300,000 faults per second, taking up ~40% of wall time spent in the kernel, and a frame hitch you can see from space. The fix is almost insultingly simple:

```c++
// setup(): read-touch every page of every alias ONCE, so demand paging
// is paid at init instead of mid-gameplay
for (auto *view : ram_views)
    for (u32 off = 0; off < RAM_SIZE; off += 4096)
        (void)*(volatile u8 *)(view + off);
```

**Find #2: excessive recompiles.** Half-Life would run fine, then crater for a second, then run fine. The JIT's code space has a size cap, and my strategy for dealing with blocks is just "drop them all and start over" if it fills. The counters I put in said that it was firing every ~110 frames, which made no sense. On Windows, MIR allocated things differently from Mac. The best fix for this was teaching MIR (in the fork) to carve all code out of one contiguous reservation, so span == bytes. Flushes per run went from 12 to 1 (I perform a flush on MMU enable), and worst frame went from 1201ms to 237ms.

The tram's average went way up by troubleshooting fairly basic structural differences between the platforms like this.

### ~570 MIPS: AICA, Again. (Part one of again.)

If you read Part 2, you already know where this is going. The flame graph's next biggest non-JIT tower was AICA audio — and its ARM7 was still running a plain interpreter, re-decoding every instruction, every execution. This didn't matter as much on the super-fast M4, but on Ryzen it was chewing up a ton of time. I'd already written a basic cached interpreter for the ARM7DI for the GBA and NDS. It just was never wired up for AICA. Wiring that up took Soul Calibur from 447 to 570 host MIPS.

### ~776 MIPS: the Part 2 optimizations, ported

Then the actual point of the exercise: bringing over Part 2's optimizations — hard block links, baked links, devirtualization — plus a pile of x86-64-specific emission work. Two highlights:

- **Materialize the fastmem base once per block.** Every access site was loading `cpu->fastmem_base` and adding, to rematerialize a value that's constant for the life of the native block. I hoisted it into a register-allocated var at block entry; every site becomes one ADD. +2.4%, SC 738 → 756.

- **Const-address folding.** Per-block constant propagation over guest registers. If I see a `MOV #imm`, MOVA, or currently-bbaked literals, I track it through copies, adds, and shifts. A data access whose address is provably a RAM constant compiles to a single displacement access, no address arithmeticor bounds check. In SC's fight loop, 53.7% of data accesses have const-RAM addresses and another 45% are the literals themselves. SC 756 → 776.

### Stretch existing MIPS a bit further

Since Part 1, "MIPS" has meant 1 instruction = 1 cycle. The real SH4 is dual-issue: two instructions retire per cycle when they pair, and the Software Manual spells out exactly when they pair: instruction groups, co-issue restrictions, per-instruction issue rates. Memory accesses also take at least 2 cycles. So I implemented it so that the emulator now charges blocks from a per-opcode attribute table, built by a script from the documentation's table. A pairing walk at block-compile time, a small memory-pressure surcharge standing in for cache/SDRAM latency, and we have a roughly correct cost model.

Games now run at the "right speed," and we get more instructions-per-MHz. Instructions per second stopped being the best metric, so from here on I use eq-MHz: emulated guest cycles per wall-clock second, divided by a million. It's the number that best indicates guest cycle throughput.

Every performance number in this post so far, and every number in Parts 1 and 2, is in the old unit. The two units happen to be within ~10% of each other on SC (dual-issue pairing and the memory surcharge nearly cancel there), so the upward trajectory is roughly the same, but our benchmark numbers are even less equivalent now.

### The economics reverse

Part 2 closed saying that per-edge optimizations were as good as they could get. Well, that only turned out to be true fro the M4. Zen 2 with the Win64 ABI is a different beast, and three of Part 2's conclusions flipped outright.

**Register residency** Part 2, step 11: pin guest R0–R6 into host callee-saved registers, +3.4% on SC. Ported it to x86-64, validated it end-to-end, A/B'd it: pin 708 vs no-pin 730. aarch64 gives MIR 31 GPRs to allocate; x86-64 gives it 11 after the JIT's own infrastructure registers, and taking two more away costs the allocator more in spills than guest-register residency saves.

**FRCHG call vs inline** The SH4's FRCHG swaps the two 64-byte FP register banks. On the M4, inlining the swap into the block beat calling a helper. But MIR had no vector IR, so the inline swap is 32 scalar 8-byte moves dumped mid-block, and on Win64's tighter register file all that inline bulk has a price. The C helper, meanwhile, is three memcpys that clang vectorizes into ~16 unrolled `movups`:

```c++
void REGS::FPSCR_bankswitch()
{
    memcpy(&fb[2], &fb[0], 64);   // clang turns this whole thing into
    memcpy(&fb[0], &fb[1], 64);   // a straight run of movups
    memcpy(&fb[1], &fb[2], 64);
}
```

The call won with a 3.7% difference. It was the same story for FTRV, the 4x4 matrix-vector transform: the inline version is ~52 scalar FP ops; a helper written so the SLP vectorizer can see the contiguous loads compiles to 4 `mulps` + 3 `addps`, and wins +4%. "Inline the hot thing" is not a law of nature.

### The light-call saga, or: measure everything, including your victories

The FRCHG result bugged me, though. Fine, the trampoline's body is better than the inline, but the call site still pays the full ABI: every caller-saved register the block has live gets spilled around the call, for a helper that touches four xmm registers and nothing else. What if the JIT could know that and issue slimmer calls to specialized functions?

So I added a "light-call ABI" to the MIR fork: a proto flagged light promises to clobber only `{rax, xmm0–5}`, and the register allocator keeps everything else live across the call. And the helpers became naked functions where the body IS the operation:

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

I retired all three a week later.

The A/Bs benchmarks never cleared the measurement noise floor. Beautiful, correct, unmeasurable code with hardcoded structure offsets and a paired proto-flag/helper contract would only be worth maintaining for a (literally) measurable gain.

### AICA, part two of again: JIT number two

Back to the ARM7. Even after the cached interpreter was wired up, the ARM7 was still around 10% of all host time on thos platform. So time to make a (basic, not super-tuned) ARM JIT!

- **ARM32 to MIR translator**, templated on CPU type so the GBA and NDS can reuse it later. I architected it with a simple, flat entry table of u32 offsets-from-code-base instead of full pointers, which would be double the size. Trying to be kind to Zen 2's cache.

This was worth +5.5% on SC over the ARM7 cached interpreter. This sounds modest until you remember that the ARM7 is maybe a tenth of host time — within its budget, that's most of the meat.

### JIT number three

The AICA doesn't just have an ARM7. It also has a DSP: a 128-step microprogram that runs *every sample*, 44,100 times a second, doing effects mixing. That's more throughput than the ARM7 has. My initial naive interpreter walked all 128 steps re-evaluating ~10 field-select branches per step — BSEL, XSEL, YSEL, ZERO, SHFT, on and on — on fields that are constant for the life of the program. Zen 2's branch predictor falls down hard on this.

So I compiled the microprogram. I specialized every step on its constant fields and emit only the taken path — the field decisions become compile-time constants and the generated code is just the arithmetic:

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

I compiled only at frame start, off the audio path, and if it's modified it gets interpreted for just that frame, because why add a bunch of compile overhead during an update? This got a huge **+8.2%** on SC.

Three JITs in one emulator now.

### Attack of the Undefined Behavior

All of the above depends on A/B runs meaning something, and I'd had a rather large noise amount between runs that made me suspicious for a while now. I built determinism checking into the bench harness ny running the workload twice, and checking the guest cycle count, the ARM7's retired-instruction count, and a final-frame screenshot hash. If they didn't all verify, it failed.

They did not match. The ARM7's instruction count swung between 109 million and 440 million per run, 4x difference, across identical runs of the same recorded workload, while the picture stayed pixel-identical.

The ARM7DI's (*not* the ARM7TDMI in the GBA/NDS) multiply timing models the Booth-algorithm early-out: cost depends on the leading zeros of the multiplier.

```c++
// before:
u32 n = __builtin_clz(multiplier) >> 1;
if (n > 16) n = 16;                       // safety clamp, right? right?
idle(n);
```

`__builtin_clz(0)` is undefined behavior, and a multiply by zero is not rare. At runtime, `bsr` with a zero input leaves the destination register unchanged, so the result comes from whatever garbage happened to be in that register. And compiler optimizations made it worse. The optimizer is entitled to assume the input is nonzero, from which it proves `n >= 0 && <= 15`. It then deletes the guard I wrote. One multiply-by-zero then charges a wildly random idle, sometimes it was up to 1.4 billion cycles. The ARM7's interrupt timing lurches, and the workload is different in every process while the game plays on unbothered.

The plain interpreter was clean because the compiler happened to keep the clamp. So the bug looked like a cached-interpreter timing divergence, and I hunted it for way too long.

```c++
// after: std::countl_zero is DEFINED for 0 (returns 32) --
// countl_zero(0) >> 1 == 16, the intended max cost.
u32 n = std::countl_zero(multiplier) >> 1;
```

I of course audited every functional `__builtin_clz` in the codebase and changed them to `std::countl_zero`, which removed the entire bug class. This flushed out another bug in the PS1 GTE.

After this, A/B's got a lot more stable, but I haven't revisited some earlier conclusions.

### Where it stands

Call it roughly 4x realtime for the 200MHz guest, on a six-core laptop from 2020, at the wall wattage of a bright lightbulb. The M4 still reads ~3x higher on the same fight, but this seems good enough. IIRC I got Half-Life up to like 560 MIPS in the tram benchmark with further optimizations. 

I did further development on a 15W version of the same chip with an iGPU, but honestly I'm getting tired of this blog post series. I've been working on Intel CPU JITs and want to start writing about those, because SH4 feels like easy-mode in comparison. 

