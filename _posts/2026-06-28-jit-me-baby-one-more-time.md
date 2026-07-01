## JIT me baby one more time

Performance. The dream of man. The age-old question: can I has the MIPS?

A while back I wrote up the theory of going from an interpreter to a JIT on the R3000. This time we're doing it for real, in C++, on the SH4, for the Sega Dreamcast. We'll head interpreter->cached interpreter->JIT->much better JIT, step by step.

### The SH4
It's a 200MHz RISC chip with 16 32-bit general registers, fixed 16-bit instructions, branch delay slots, an FPU with a couple of fun mode bits, and an MMU we avoid emulating if possible (but we'll get to how I do it, maybe in a later article.) By the end we'll have a dynarec that beats a very, very fast cached interpreter.

#### A note on MIPS
For SH4, I use 1 instruction = 1 cycle, strictly. I may change this in the future, but it's been that way since the beginning.

Also, the MIPS numbers I'm sharing are from booting Soul Calibur past the BIOS and into the attract mode for a few thousand total frames. They're approximate, and heavy gameplay could (and definitely did) bring them down, but they measure progress by being a repeated benchmark.

### 100 MIPS. Where we started: a plain interpreter

My SH4 interpreter wasn't anything special. It just fetched and executed opcodes. An interesting aspect of it was that for delay slots, it simply called fetch_and_exec() from inside the function body, thus side-stepping having to worry about them, and updated the PC appropriately afterward. This did become a bit of a wrinkle later on, but eventually worked out.

One "fun" part of the SH4 is that some instructions change what they do based on some bits in the status register.

So let's get to naive interpreter...

### Step 1: a very naive interpreter

SH4 instructions are all 16 bits. The top nibble usually picks the family, and the rest of the bits are some mix of register fields (`n`, `m`), a displacement (`d`), or an immediate (`i`). So `0110nnnnmmmm0011` is `MOV Rm, Rn`, and you pull `n = (op >> 8) & 0xF`, `m = (op >> 4) & 0xF`.

```c++
void SH4::SH4_MOV(u16 op) {
    u32 n = (op >> 8) & 0xF;
    u32 m = (op >> 4) & 0xF;
    R[n] = R[m];   // the actual work
}
```

### Step 2: the interpreter loop

Your plain regular interpreter loop looks like this.

```c++
while (cycles_left > 0) {
    u16 op = mem_read16();
    PC += 2;
    decode_and_execute(op);   // big switch or a table
    cycles_left -= cycle_cost(op);
}
```

On a naive interpreter, every single time you execute that instruction, you're paying multiple costs that we can eliminate.

1. fetch the instruction - eliminated by CI
2. decode the instruction family - eliminated by CI
3. setup the stack and call the function for the handler - eliminated by CI, eventually
4. decode the instruction operands - eliminated by CI
5. actually execute the instruction's logic
6. account for PC changes and cycle counts - eliminated by CI, eventually

### ~230 MIPS. Step 3: the cached (block) interpreter

The idea is exactly the R3000 one: decode a run of instructions once, store the decoded form, and execute from the decoded form. On the SH4 a decoded "cell" looked roughly like this:

```c++
struct cell {
    void (SH4::*handler)(cell*);   // pre-selected handler (a member function)
    u16  opcode;                   // the raw instruction, kept around
    u32  n, m;                     // pre-extracted register fields
    u32  imm;                      // pre-computed immediate/displacement
    bool decoded;                  // has this slot been translated yet?
};
```

The handler no longer re-derived its operands from the opcode; the block translator did that once, up front, so the handler just read the pre-decoded fields off the cell:

```c++
void SH4::SH4_MOV(cell *c) {
    R[c->n] = R[c->m];   // operands already extracted
}
```

We translated these in groups called blocks from some PC, stopping at the first control-flow instruction (plus its delay slot), and stashed the block in a map keyed by the physical RAM address it came from. Next time the PC landed there, we skipped fetching+decoding entirely.

That already knocked out costs (1), (2), and (4) from the list above. But there was another easy win sitting right there. On a 64-bit host the cell was a bloated ~40 bytes, and every dispatch called through a pointer-to-member. Both were quietly costing us, and both were fixable.

Now, fetching a block to execute looked like this:

```c++
block *get_block(SH4 *cpu, u32 pc) {
    if (block *b = block_store.get(phys(pc))) return b;
    return block_store.put(phys(pc), translate_block(cpu, pc));
}

while (cycles_left > 0) {
    block *b = get_block(cpu, cpu->PC);
    b->run(cpu);
    cycles_left -= b->cycles;
}
```

So the basic idea was, each time it's time to execute, we try to get a block. The get block function checks if the block exists and is valid and returns it if so, or compiles and returns it if not.

#### Compiling blocks
I basically did what we did in the original post. `translate_block` started at the block's PC and walked forward, turning one instruction at a time into a cell: look up the handler, pre-extract the operand fields, pre-compute the immediate, append the cell, advance. It kept going until it hit a stopping condition — a branch (plus its delay slot), an FP-slice change, or the safety cap (see Block limits below) — then recorded the block's total cycle cost and handed it back. It ran exactly once per block, so a slightly high initial cost was repaid when every later execution just replayed the fetched-and-decoded cells.

##### Block limits
I used branches and some state-change instructions to end a block, plus an arbitrary safety cap so a straight-line run doesn't translate forever - 256 instructions.

Because the SH4 has some instructions that differ in behavior (and even validity) based on some bits in the status register, my cached interp selected each cell's handler depending on the current FP mode. I specialized a block on the "slice": `slice = (SZ << 1) | PR`, two bits out of `FPSCR`. A block compiled for one slice couldn't be reused under another, so anything that changed the slice ended the block. 

#### Storing blocks
Block lookup was the single hottest lookup in the emulator, so an `std::map` or `std::unordered_map` was out. The store was a two-level sparse page table indexed straight off the block's physical RAM offset. 

The key was the 24-bit offset of the block's first instruction, and since SH4 RAM is 16 MB and instructions are 2-byte aligned it split cleanly in two: `off >> 12` picked one of 4096 per-4KB-page slot arrays (each one lazily allocated the first time code ran on that page, so the table stayed sparse), and `(off & 0xFFF) >> 1` picked the 2-byte slot within it. A lookup was two array indexes and a null check.

Each block was basically just a list of instructions. At first, these were just pointers to the decode table. It also included a cycle count. 

#### Invalidating blocks
What about SMC, Self-Modifying Code? Or what about when new code is loaded and we need to invalidate the code that was replaced? I borrowed a 4K page-invalidation scheme from my NDS cached interpreter. I kept a version counter per 4K page; when I compiled a block I stamped it with the current version of the page(s) it covered, and when the CPU wrote into a page that held code I just bumped that page's counter. At the next block lookup, `get_block` compared the stamped version against the page's current one, and a mismatch meant "throw it away and recompile."

That's what made it lazy and cheap: a store into a code page was a single counter bump, and stale blocks quietly failed their version check the next time the PC reached them. However, a write anywhere in a 4K page invalidated every block on that page, even the ones the write didn't touch. This is a mistake I rectified later.

### ~280 MIPS! Step 4: static dispatch

Look again at how our handler was declared:

```c++
void SH4::SH4_MOV(cell *c) { ... }
```

Storing a pointer to that means storing a pointer-to-member-function, which in C++ is 16 bytes, because both the object and the function are represented, 2 pointers. Worse, calling through one isn't a plain indirect call: the compiler emits a hidden branch on the low bit of the pointer to decide "virtual thunk or direct function?" On every dispatch, on the single hottest path in the whole emulator.

So I made them into plain static functions that took the CPU as an argument explicitly:

```c++
static void SH4_MOV(SH4 *cpu, cell *c) {
    cpu->R[c->n] = cpu->R[c->m];
}
```

and store a plain function pointer:

```c++
void (*handler)(SH4*, cell*);   // 8 bytes, one direct call, no branch
```

The handler field was now 8 bytes instead of 16, and dispatch became a single indirect `call` with nothing in front of it. Passing `cpu` in cost nothing — it was pinned in a register across the whole run anyway.

### ~325 MIPS! Step 5. Shrink the cell

With the fat pointer-to-member gone, the cell was shrinking, and cache friendliness is important. Dispatch was one linear walk over these cells, so their size directly decided how many fit in a cache line.

The naive cell reached for `u32` on everything out of laziness. But an SH4 register index was 0–15, so `u8` was plenty; every displacement and immediate fit in 16 bits; and the `decoded` flag was redundant, because a null `handler` already meant "not translated yet." So I packed it down and pinned the alignment:

```c++
struct alignas(16) cell {
    void (*handler)(SH4*, cell*);   // 8
    u16  opcode;                    // 2
    u8   n, m;                      // 2
    i16  disp;                      // 2
    u16  imm;                       // 2
};   // exactly 16 bytes, four to a cache line, never straddling
```

So, each compiled instruction now cost only 16 bytes, fitting neatly even in an older cache line. 

Smaller is not automatically faster, however. I later tried squeezing the cell to 8 bytes by swapping the 8-byte handler pointer for a `u16` index into a table of handlers. It came out slower — the extra dependent load (index → table → function pointer → call) cost more than the density saved. 16 bytes was the sweet spot. Measure everything; intuition lies.

### ~335 MIPS! Step 6: store cells by value

As you may recall, each block was a list of pointers to the decode table. That indirection was the next thing to go. Holding `cell*` pointers meant every dispatch chased a pointer into a scattered decode table, and the cells weren't adjacent, so there was no way to walk them linearly.

So I stored the cells by value, contiguously, in the block itself:

```c++
struct block {
    std::vector<cell> cells;   // by value, no longer cell*
    u32 cycles;
};
```

On its own it was only worth a few percent, but it's the change that made the next step possible at all.

### ~390 MIPS! Step 7: tail-threaded dispatch (musttail)

Next step, delete the dispatch loop entirely! Instead of each handler returning to a `while` loop that fetches the next cell and calls it, each handler ends by calling the next handler directly — and tells the compiler to make it a real tail call:

```c++
static void SH4_MOV(SH4 *cpu, cell *c) {
    cpu->R[c->n] = cpu->R[c->m];
    cell *n = c + 1;
    [[clang::musttail]] return n->handler(cpu, n);   // jump, don't call
}
```

`musttail` guarantees the call compiles to a `jmp`, not a `call` — no return, no stack growth, no loop bookkeeping between instructions. The run loop just kicks off `cell[0]`, and control flows handler to handler straight down the block. When a handler needs to stop because it's the end of a block or some state changed that necessitates it, it just returns, and we land back in the outer loop.

One additional thing though. Anything that can't have a contiguous "next cell" — a branch delay slot, a step into non-RAM — uses a plain non-chaining variant of the handler that returns instead of jumping. Worth about 18%.

### ~400 MIPS! Step 8: preserve_none

With the chain in place, the calling convention itself was the overhead. Every one of those tail calls still honored the platform ABI — save and restore the callee-saved registers — even though the only things any handler cared about were the same two or three values (the CPU, the cell pointer, the cycle count) flowing straight down the chain.

`[[clang::preserve_none]]` to the rescue. No callee-saved contract, so the VM state just stays in registers across the whole block. In theory, a big win.

In practice it barely moved the needle at first. Every guest load and store still called out to the big `mem_read`/`mem_write`, and `preserve_none` has to spill all those freshly-pinned registers around any normal call. So the state I'd just pinned got shoved back to memory on every single memory access. It looked like a dud, until the next step fixed exactly that.

### ~460 MIPS! Step 9: the inline fast-RAM path

`mem_read`/`mem_write` were each one big function handling everything — the main-RAM window, on-chip registers, the operand cache, store queues, bus callbacks. So every guest access was a function call, and under `preserve_none` that call spilled the registers I'd worked to pin.

But the majority case is simple: a plain read or write of the RAM. So I split the accessor into an `always_inline` fast path for that case, folded right into the handler, and a `noinline` cold path for everything else:

```c++
[[gnu::always_inline]] u32 read32(SH4 *cpu, u32 addr) {
    if (in_ram_window(addr)) return *(u32 *)(cpu->ram + (addr & RAM_MASK));  // folded inline
    return read32_slow(cpu, addr);   // MMIO / OC / SQ / bus -- kept out of line
}
```

Now the hot path had no call in it at all — nothing to spill. And since we were doing `preserve_none,` the fold was worth about +6% under plain musttail, but about +10% under `preserve_none`.

### ~540 MIPS! Step 10: block linking

This next was the single biggest step in the whole cached interpreter: about +17%. Things were fast inside each block, but between blocks needed improving. Every branch still dropped back to the run loop, which re-masked the PC and did the two-level table lookup to find the next block.

So I linked the chain across block boundaries too. The block grew two successor slots:

```c++
struct block {
    std::vector<cell> cells;
    u32 cycles;
    block *link_next, *link_alt;   // two most-recent successors
};
```

When a branch handler has computed its target PC, instead of returning it resolves the successor block and tail-jumps straight into that block's first handler. The lookup is a cached-hit check first, falling back to the real lookup only on a miss:

```c++
block *link_fast(SH4 *cpu, u32 offset, u8 slice) {
    block *cb = cur_block;                                 // block we're branching out of
    for (block *s : { cb->link_next, cb->link_alt })       // two most-recent successors
        if (s && s->start_off == offset && s->valid && s->slice == slice)
            { cur_block = s; return s; }                   // hit: no lookup at all
    return link_slow(cpu, off, slice);                     // miss: table lookup, cache it
}
```

Two slots is enough to cover both a direct branch and a repeated indirect one bouncing between a couple of sites. So the tail of a branch handler became:

```c++
// ... target PC already in cpu->PC ...
if (event_due(cpu) || !in_ram(cpu->PC)) return;    // let the scheduler run; don't chain out of RAM
block *n = link_fast(cpu, phys(cpu->PC), slice(cpu));
cpu->base_pc = cpu->PC;
cell *i0 = n->cells.data();
[[clang::musttail]] return i0->handler(cpu, i0);   // jump straight into the successor
```

### ~595 MIPS! Step 11: block cycle batching

Even fully chained, we were still doing per-instruction bookkeeping: every cell decremented the cycle counter and checked whether the budget had run out. But a block's total cycle cost is known at compile time. So I charged the whole thing up front .

```c++
// before -- every straight-line step pays the counter:
#define DISPATCH_NEXT   if (--cpu->cycles <= 0) return;      \
                        cell *n = c + 1;                     \
                        [[clang::musttail]] return n->handler(cpu, n);

// after -- the run loop charged block->n up front, so the step is a bare jump:
#define DISPATCH_NEXT   cell *n = c + 1;                     \
                        [[clang::musttail]] return n->handler(cpu, n);
```

The budget-and-interrupt check now happened once per block boundary instead of once per instruction:

```c++
cur_block = b;
master_cycles += b->cells.size();   // charge the whole block, once
cell *i0 = b->cells.data();
i0->handler(cpu, i0);               // chain runs
```

Once a block starts, it runs to the end, so an interrupt or timer can theoretically go unnoticed up to a block late. On the Dreamcast that's fine.

### ~610 MIPS! Step 12: lazy PC calculation

The cached interpreter still wrote the PC back after every instruction (`PC += 2`), even though nothing needs it mid-block except PC-relative instructions, and the block exit. So I stopped maintaining it per-instruction and calculated it only where it was actually read.

There's no room in a 16-byte cell for a new field, and no need for one either: a cell that reads the PC doesn't use its immediate/displacement, and a cell that uses those doesn't read the PC. So I overlapped them in a union.

```c++
struct alignas(16) cell {
    void (*handler)(SH4*, cell*);   // 8
    u16  opcode;                    // 2
    u8   n, m;                      // 2
    union {                         // 4
        struct { i16 disp; u16 imm; };   // operand (most cells)
        u32 baked_off;                   // compile-time-baked PC / RAM offset
    };
};   // still exactly 16 bytes
```

When a block is compiled, each PC-reading cell gets its byte-offset-into-the-block written into `baked_off`. The run loop stashes the block's entry PC once, and the read becomes two loads.

```c++
// run loop / link, once per block:
cpu->base_pc = cpu->PC;

// a straight-line handler no longer touches PC at all; only the cells that
// actually read it reconstruct it from the baked offset:
#define CUR_PC   (cpu->base_pc + c->baked_off)
```

### ~625 MIPS! Step 13: pointer-baked literal loads

The SH4 loads constants with `mov.l @(disp, PC)` — a PC-relative read from a literal pool sitting next to the code. In an attract-mode histogram those were about 13% of instructions executed, and their address is fixed at compile time.

So I resolved it once, at compile time. Using the baked offset from the last step, I work out the literal's fixed RAM address; if it stays in RAM without wrapping the window, I swap in a specialized handler and stash the address in the cell:

```c++
// in translate_block, for each `mov.l @(disp,PC)`:
u32 lit = (cell_off & ~3u) + disp * 4 + 4;   // its address is fixed forever
if (lit < RAM_SIZE) {
    c.baked_off = lit;                        // where the literal lives
    c.handler   = &SH4_MOVLI_RAM;             // specialized reader
}
```

The specialized handler then skips the PC reconstruction, the address arithmetic, and the bounds-check branch entirely.

```c++
static void SH4_MOVLI_RAM(SH4 *cpu, cell *c) {
    cpu->R[c->n] = *(u32 *)(cpu->ram + c->baked_off);   // live read off RAM
    DISPATCH_NEXT;
}
```

One thing I deliberately did not do: bake the literal's value. Only the address is frozen, and that genuinely is constant. SH4 code tends to use PC-relative variables a bunch, probably due to the 16-bit instruction size limiting where it can store globals easily.

### Why not just JIT at this point?
At this point I wasn't sure why I wasn't just working on a JIT. It felt like the optimizations I was making were at that level anyway. So that's what I did next.

I did end up speeding the CI up a bit more incidentally with improvements made for the JIT. For instance, it turns out you don't even NEED the code block invalidation I naively ported from my NDS cached interpreter. But that's a story for Part 2: Oops, I JIT'd again.