## Retro Speed Boost!

When emulating an old system cycle-accurately, there's a few ways of doing it.

One of the problems we run into, in terms of efficiency, is how do we deal with clock dividers?

For instance, in the Sega Genesis, there is a master clock, and the following components and dividers:

* the VDP, 4 or 5 (it changes)
* the M68K, 7
* the Z80, 15
* the ym2612, 144
* the psg, 220

There's a simple way to do this:

```c
cycles_til_vdp--;
cycles_til_m68k--;
cycles_til_z80--;
cycles_til_ym2612--;
cycles_til_psg--;

if (cycles_til_vdp <= 0) {
    cycles_til_vdp += current_vdp_divisor;
    tick_VDP();
}
// and 4 more if's like that
```

You can speed this up a LITTLE by doing:

```c
step_size = MIN(cycles_til_vdp, MIN(cycles_til_m68k, MIN(cycles_til_z80, MIN(cycles_til_ym2612, cycles_til_psg))));

cycles_til_vdp -= step_size;
//etc.
```

But doing it this way was literally taking up about 30% of my frametime for Sega Genesis, leaving me at around 80FPS on my Macbook Air M2.

### How can we improve? A typical scheduler?
Schedulers are commonly used to speed up newer systems. OK, so in each frame, there's 896,040 cycles (~53MHz master clock). Every 4 or 5 the VDP is ticked, so in worst case, that's scheduling at LEAST 896,040 / 4 = 224,010 events per FRAME.

That did not sound like a good idea. Even if I just did it every scanline, that'd be at least ~1.4k events per scanline. Maybe doable, but doesn't seem that great an idea.

### A pre-computed scheduler?
My next thought was, what if I could pre-create a schedule and use it over and over?

To do this, the schedule's cycle length would have to be the least common multiple of all of the dividers. So let's see. Least common multiple for 4, 7, 15, 144, 220: 55,440. That's a lot...doesn't sound like it'll easily fit in cache. Also, it isn't evenly divisible by a scanline. If I add the number of cycles in a scanline to that calculation, I get 308 scanlines at 1,053,360 cycles!

I was able to get it down to 7 scanlines/23,940 cycles by removing the PSG and ym2612 from the equation, planning to schedule them normally. Unfortunately I still can't schedule per-scanline this way, and games can (theoretically, at least) change the VDP divisor between 4 and 5 mid-line.

### A note about schedulers here
When I said "planning to schedule them normally," I am assuming you are familiar with emulator schedulers.

The typical algorithm for an emulator scheduler is basically:
```c
while(true) {
    event = first_event;
    while(event->timecode <= current_clock) {
        event->callback();
        delete_event(event);
        event = first_event;
    }
    
    run_block(); // this will advance the clock by some regular # of cycles
}
```

Where a "block" is a number of cycles of variable size. Typically run_block() is used to run block(s) of JIT code, but can be used for a regular interpreter as well.

### The solution!
When the solution hit me, it felt so obvious!

We've got 3 components we're mediating between: the m68000, the z80, and the VDP. What we can do instead is...

* always move the clock forward by VDP cycles (4 or 5)
* our m68k and z80 "cycles til next tick" will always be in the range of 1-15 for the z80, or 1-7 for the m68k
* Since we're always moving forward by either 4 or 5, that means that there's only(15*7)=105 possible states for these to be in. 

At any point in our 105*2 entry table, any of the following 5 things can happen:

```c

M68K_tick(); cycles_til_next_m68k += 7;
Z80_tick(); cycles_til_next_z80 += 15;
VDP_tick();

// or
Z80_tick(); cycles_til_next_z80 += 15;
M68K_tick(); cycles_til_next_m68k += 7;
VDP_tick();

// or
M68K_tick(); cycles_til_next_m68k += 7;
VDP_tick();

// or
Z80_tick(); cycles_til_next_z80 += 15;
VDP_tick();

// or
VDP_tick();
```

I could easily represent this as a function pointer to one of 5 functions.

I also realized that the "output" cycles_til_next_z80/m68k depended totally on the input. So if I included a part in the table that points to the next index, and made sure I got the ordering correct, I wouldn't even need to keep track of "cycles til next tick" for any component at all. 

I end up with this data structure:
```c
struct genesis;
typedef void (*gensched_callback)(struct genesis *);

struct gensched_item {
    u32 next_index;
    gensched_callback callback;
};

struct genesis {
    struct scheduler_t scheduler;
    struct gensched_item scheduler_lookup[105 * 2];

```

I fill it like this:

```c
static void create_scheduling_lookup_table(struct genesis *this)
{
    u32 lookup_add = 0;
    for (i32 cycles=4; cycles<6; cycles++) { // 4 and 5 are the two possible vdp divisors
        for (i32 tbl_entry = 0; tbl_entry < 420; tbl_entry++) {
            // Decode values from tbl_entry
            i32 z80_cycles = (tbl_entry % 15) + 1;
            i32 m68k_cycles = (tbl_entry / 15) + 1;

            // Apply cycles
            z80_cycles -= cycles;
            m68k_cycles -= cycles;

            struct gensched_item *item = &this->scheduler_lookup[lookup_add + tbl_entry];

            // Determine which function to use
            if ((z80_cycles <= 0) && (m68k_cycles <= 0)) {
                if (z80_cycles < m68k_cycles)
                    item->callback = &c_z80_m68k_vdp;
                else
                    item->callback = &c_m68k_z80_vdp;
                z80_cycles += 15;
                m68k_cycles += 7;
            }
            else if (z80_cycles <= 0) {
                item->callback = &c_z80_vdp;
                z80_cycles += 16;
            }
            else if (m68k_cycles <= 0) {
                item->callback = &c_m68k_vdp;
                m68k_cycles += 7;
            }
            else {
                item->callback = &c_vdp;
            }

            // Encode values of next to tbl_entry
            item->next_index = ((m68k_cycles - 1) * 15) + (z80_cycles - 1);
        }
        lookup_add += 105;
    }
}
```

And instead of all those ifs checking if things should run or not, subtracting all those counters separately, etc., the inner loop looks like this:

```c
static void genesis_step_loop(struct genesis* this)
{
    u32 lu = 105 * (this->clock.vdp.clock_divisor - 4);
    this->scheduler_lookup[lu + this->scheduler_index].callback(this);
    this->scheduler_index = this->scheduler_lookup[lu + this->scheduler_index].next_index;
    this->clock.master_cycle_count += this->clock.vdp.clock_divisor;
}
```

The next state is implied by the current one, so this just goes through and  calls the right function that'll do things in the right order, and sets the next state.

### The result: performance!
My Genesis emulator went from ~80fps up to 150fps. Not quite a 2x speed boost, but really significant nonetheless! And I haven't sacrificed ANY accuracy*

Furthermore, my emulator is now MUCH more scheduler-friendly. It could easily run multiple entries from this lookup table in a row during "scheduler blocks," or the blocks could just always be the size of the VDP divisor.

### Problems
There are still two problems left, however. We never handled sound, and we did lose a LITTLE accuracy.

#### We've got no sound!
My plan is to convert my Genesis emulator to run on a typical scheduler, and schedule the next updates for YM2612, YM2612 envelope, and SN76489 one scanline at a time, since they have such huge divisors. I could probably even do a frame at a time. This will only be about 45 events per scanline, which is much more in-line with the usage of other emulator cores and I know it will perform well.

#### There is one slight inaccuracy
But this one is easy to fix!

The problem is that the clock always moves in jumps of 4 to 5. So the Z80 and M68k tick functions have the wrong master clock, off by up to 4 ticks. They do consider various things, such as if they should be un-paused, using this clock.

This is actually pretty easy to solve. If I add a *little* more state to the struct:

```c
struct gensched_item {
    u32 next_index;
    gensched_callback callback;
    
    u32 ticks_add_z80;
    u32 ticks_add_m68k;
    u32 ticks_add_vdp;
};
```

I can fill this in inside the function that makes the look-up table, and each component will have the correct # of clocks when it is its turn to tick, while still having very low overhead.

### The End
This technique should generalize to most retro systems and make deciding which order to do ticks in super fast. I'll be adding it to my other cores soon.

I hope this is inspirational if you're looking to increase the speed of your own retro emulator with tightly coupled components and high accuracy.