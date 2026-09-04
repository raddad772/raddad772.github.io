## Oops, I JIT'd again!

In Part 1 we took a plain SH4 interpreter from ~100 MIPS to a ~625 MIPS cached interpreter. By the end of it I was baking addresses into cells and hand-threading dispatch — doing a lot that a compiler does except actually emitting code. Time to emit the code.

I start this with the same benchmark as before: Soul Calibur, boot past the BIOS into attract mode, a few thousand frames. Same 1 instruction = 1 cycle rule as last time. Numbers are still approximate.

Around the time I hit 1400 MIPS on the SC benchmark, I started benchmarking actually in-game which dropped it quite a bit, so I've included adjusted numbers accordingly. With that said, although line go up, a few of the numbers are a bit approximate.

### The plan

I didn't write my own assembler backend, but used [MIR](https://github.com/vnmakarov/mir). It's basically intended for specifically this use, though I did find a few hard edges. I emit MIR's IR for instructions (using its built-in functions), and it hands back a function pointer. Most things from Part 1 stay. The same blocks, block store, and invalidation (for now). At first, blocks simply gain native pointers.

Cold code is still cached-interpreted, but hot code gets compiled. The two tiers share everything. Easy, right?

### Step 1: the naive JIT (~353 MIPS)

My first attempt was thirty percent SLOWER than the cached interpreter.

```c++
block *b = get_block(cpu, cpu->PC);
if (b->native)            b->native(cpu);           // tier 2: compiled
else if (++b->hot > 50)   b->native = jit(cpu, b);  // hot enough, compile it
else                      run_cached(cpu, b);       // tier 1: Part 1's interpreter
```

Although blocks were native, block-to-block links were not. Each native block ran, returned to C, and the C code masked the PC, looked up the next block, and called back into native. Due to the average block size of around 8 instructions, that happened about 25 million times per second.

### Step 2: make it worse (~280 MIPS)

I restructured the driver so every block bounced through a little router that called a C helper to find the next block. Still a C call per block, now with extra steps: ~280 MIPS. Going the wrong way!

The lesson wasn't "route better." It was "eliminate native->C transitions."

### Step 3: block linking, again (~660 MIPS)

Blog post Part 1 step 10 again. Each block now caches its hot successors; a branch checks the cache and jumps straight into the successor's code, no `block_store.get()`, no C:

```c++
block *s = cur_block->link_next;
if (s && s->start_off == off && s->slice == slice && s->native)
    s->native(cpu);         // hit: straight into the successor
else
    s = jit_link_slow(...); // miss: real lookup & cache it for next time
```

That was finally enough to beat the interpreter, barely.

### Step 4: in-MIR dispatch (~765 MIPS)

That successor check was still a C function call at the end of every native block. The hot edge kept leaving generated code.

So I emitted the whole fast path as MIR — the successor check and the jump are machine code inside the block itself. Now a hot loop of linked blocks is one long run of native code that never touches C code (outside of helpers). +20% over the cached interpreter, and the first time the JIT felt like a JIT.

### Step 5: the problem wasn't the SH4 (~1150 MIPS)

Since eliminating C call-outs was important, I took a look at what was causing them, and it was scheduled events.

The AICA's ARM7 sound CPU was single-stepped one instruction per scheduler event, and accounted for 97% of ALL scheduler events. This sliced the SH4 into an ~80–130 cycle chunks before yielding. My C-free native chains were being interrupted constantly to run a single ARM instruction.

There is a tax every time you need to go to C: you need to spill your registers to the stack, and reload them after. The more C transitions, the harsher the tax, which is why this made such a huge difference when I fixed it.

I fixed it by batching the AICA. Run it in bigger chunks, resync less often, and we jump to ~1150.

You'll note that these steps seem big, but increases compound. A 1% increase from 100->101 isn't big, but a 1% increase from 1000->1010 is bigger.

### Step 6: delete the invalidation scheme (~1215 MIPS)

Remember the 4K page-version SMC scheme I ported from my NDS interpreter in Part 1?

The SH4 has to invalidate its instruction cache when code changes. So instead of stamping pages on every write and re-checking versions on every dispatch, I just re-verify blocks (a quick hash of the source opcodes) when the guest touches `CCR.ICI`.

That deleted the per-write bookkeeping and about 10 instructions from every block boundary for +15–20% perf. I also made it so the ONLY exit condition between blocks was "current time >= next_event," a field the scheduler now keeps current to the next event.

### Things I was sure would work, and didn't

- **Cross-block register residency.** Keeping the hottest guest registers pinned in host callee-saved registers across blocks got ~0%. At the time, the ARM7 was still chopping SH4 execution into ~100-cycle slices - this would become a win later after I addressed that. 
- **JSR devirtualization, store-queue specialization.** ~0% on this workload

I tried a bunch of other things I don't even remember.

From here on the benchmark changes. The numbers below are a real Soul Calibur 1v1 with player-controlled gameplay driven by a new input recording and playback system. These numbers are generally lower than the attract-mode numbers from before. I measured both ways of doing it back-to-back on the current build: the old benchmark read 1673 MIPS where the fight reads 1215, so I'm using a ratio of 0.73.

The steps below quote percentages instead of MIPS — the fight harness didn't exist yet while they were being built, so each feature's number is an A/B of just that lever on today's build.

### Step 7: block fusion — a lot of effort for just a little reward (+8% SC, +7% Ikaruga)

Normally I end a block on a control-flow instruction. But, what if I don't? What if I profile hot paths and join blocks together?

Nothing, is what. Slowness. Register pressure and frequent C call-outs killed this.

One sub-experiment did pay off for some games though. On SH4, it's a common idiom to JSR to an RTS;NOP block, maybe it saves code size? IDK. When a JSR;RTS;NOP keeps gappening, I emit a compare-and-direct-jump for the hot target ahead of the generic path. About half of all block exits in SC's 3D loop are JSR/RTS,NOP:

```c++
// terminal of a block ending in JSR @Rn — target observed at compile time
if (regs.PC == 0x8C0104E0)   // the address this site called last time
    goto block_8C0104E0;     // direct jump, no lookup
// fell through: target changed, take the generic linked path
```

This got like +8% on Soul calibur and like +7 on Ikaruga, but many games were just a wash with it.

### Step 8: hard block linking (+10%)

Step 4 brought me block-to-block hops in native code, but it was still an indirect jump through a cached pointer plus a handful of guard compares. For block edges whose target is a fixed address, conditional branches included, I can do better: guard, then jump directly into the successor's body, no pointer chase at all. ~94% of static edges qualify.

```c++
// block terminal, emitted as MIR. slot[] is step 4's cached-successor scheme.
if (pc == taken_pc          // which way did the branch commit?
    && !SR.FD               // target may owe an FPU-disable exception
    && edge_gen[0] == link_reset_gen
    && target->native_gen == edge_native_gen[0]) {
    cur_block = target;     // both immediates at compile time
    goto taken_block_body;  // straight into the successor
}
// any guard fails: fall through to step 4's slot path, then the driver
```

One note on this code: I know it's a lot more guards than needed, I addressed that later.

And this did leave one very annoying-to-troubleshoot bug. It created a new class of invalidation bug. The step 6 invalidation scheme re-verifies a block when the guest signals `CCR.ICI,` but a stale predecessor can now jump straight into a block's old native code without ever passing a verification site. I found this out because a texture goes black in one game, once, minutes in. Fix: invalidation unlinks a block's incoming edges too, and the generation counters gate every baked entry.

### Step 9: bake the pointers (+3.5% literals, +1.2% links)

With links being direct jumps, the remaining per-edge work is in loads: the successor's block pointer, its function pointer, and literal values the block reads from RAM. All of them are usually constant. So I baked them as immediates when PC-relative on the same page. The literal's page gets write-protected, and on fault the literal gets un-banked. 

```c++
// guest: mov.l @(0x14,PC),r2    ; loads 0x8C012345 from the code's literal pool
// before: r2 = read32(0x8C0104F8);   // a C-trampoline and memory access every time
// after:  r2 = 0x8C012345;           // immediate; our write-protected page defends it
```
### Step 10: delete the guards (links go +10% → +15%)

At the end of step 8 is some pretty bad code. Four compares per edge, and three of them exist to answer "could the target have changed since I linked?" A guard is paid on every traversal, forever, against an event that almost never happens. The better answer for rare events is eager invalidation: make the rare event itself fix the links.

- reset-generation guard: gone; a cache flush walks the blocks and severs links.
- target-recompiled guard: gone; every block keeps a list of incoming edges, and replacing its native code unlinks them.
- FPU-disable guard: gone; the write to SR that sets FD severs links right there.

```c++
// the whole exit now:
if (pc == taken_pc && cycles_left > 0) {
    cur_block = target;
    goto taken_block_body;
}
```

Invalidation events now scan over blocks, etc., so this only works because steps 6 and 9 already made those events rare and cheap to detect.

### Step 11: register residency, redeemed (+3.4% SC, +5.8% Ikaruga)

Since step 5, the ARM7 interrupts us a LOT less. This means a LOT less spills back to C each frame. The residency idea from the failed-experiments list deserved a proper retry. I tried guest R0–R6 pinned in host callee-saved registers across block boundaries.

Too much debugging later, the crashes were not my bug: MIR's optimizer had two latent bugs exposed by registers that stay live across its call boundaries. The coalescer would merge a temporary into a global tied to a call-clobbered register, and cross-function jumps weren't treated as call-like barriers in liveness analysis, so "dead" pin reloads got eliminated. I fixed both in my MIR fork.

With the tooling fixed, guest R0–R6 live in host x22–x28 for the whole run. (Remember, I'm developing on Mac M4, which is AArch64). Blocks read and write them as registers, and only the C boundary touches memory.

+3.4% on SC, +5.8% on Ikaruga. Pinning more registers (up to 16, spilling into caller-saved ones) measures slower. The save/restore brackets around every C call still outweigh the residency.

### Step 12: turn it all on (+32%, ~1400 MIPS)

Each lever since step 7 was developed in isolation behind a flag and measured on its own. Step 12 is flipping them all default-on and measuring per-game. SC +32%, Jet Grind +21%, Dino Crisis +17%, Sonic Adventure +16%, Ikaruga +15%, Ecco +6%, Rez +5%. And they compound: chaining the individual percentages above projects ~1200 MIPS, but the fight measures ~1400. The composition beats the sum of the parts because the features feed each other — link baking's occasional losses disappear once residency relieves register pressure, etc.

### An aside: the MMU

Most Dreamcast games never turn the MMU on, so neither do I. The Windows CE games do, and Half-Life especially was a big target for me. I was developing the MMU in a separate branch, so porting these over to the MMU branch was a nice speed boost, from ~670->820 MIPS on the tram ride benchmark I'd been using.

### Where it stands

The Soul Calibur fight benches ~1400 MIPS (1270–1450 run to run; an interleaved session reads ~1215). On the normalized scale that's roughly 3× the Part 1 cached interpreter and ~19× the plain interpreter the emulator started at. That's 7× real-time for an emulated 200MHz chip, and about 3.1 host cycles per guest cycle.

One caveat on all these percentages: this machine is an Apple M4, which eats indirect branches and dependent loads for breakfast. 

The whole MMU saga is its own post, but honestly, I've more than hit my Dreamcast perf goals. I did a whole series of optimizations when porting to x86-64, and on a decade-old mid-range Ryzen I can get like 400 MIPS in Half-Life now and like 800+ in Soul calibur. I'm working on Intel emulation lately, and that JIT has been a whole harder beast with interesting MMU fun. I'll probably write part 3 about the differences in JIT design I've come to, as well discuss in-depth MMU emulation there.