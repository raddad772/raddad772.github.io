## The inflection point of emulation accuracy

Ever wondered why you don't see JIT used for NES? Other than the relative simplicity of the system, there's a reason for that.

In the world of game consoles (and, to a lesser extent, computers), there is a point where things radically changed. Before this time, things were mostly done one way; after this time, they were done another. 

This change has a lot of bearing on how you build an emulator. Systems from the old way should be built one way for best compatibility; systems after this point are generally a lot more tolerant of inaccuracies.

I call this point, the inflection point of emulation accuracy.

### Grouping systems
It's not a perfectly clean divide. Many systems very cleanly came before or after, but some came at an awkward point in history where things still weren't settled yet.

These groups shared some broad similarities.

| Characteristics of older consoles                                   | Characteristics of newer consoles                                                |
|---------------------------------------------------------------------|----------------------------------------------------------------------------------|
| Games were generally hand-written in assembly                       | Games were often written in higher-level languages such as C                     |
| Timing was mostly based around the scanning of a CRT                | Separate hardware timers and RTCs were available, making timing much more flexible |
| Rendering an image always happened once per CRT frame in fixed time | Rendering an image could take a variable amount of time                          |
| Address and data busses were simple and immediate, there were few or no caches | Front-Side Busses became the norm, as did caches |

### Details
Games written for older systems were generally a lot less flexible. Their cycle counts were sometimes adjusted manually to make them take exactly some portion of a scanline to execute. At times, they rely on super-obscure behaviors of the hardware, or their bugs require accurate emulation of aspects of the hardware such as a floating bus. They were timed based on how the CRT operated, and had to have pixels out at specific intervals. Getting things wrong can mean wrong graphics, slow or fast gameplay, weird bugs, crashes, and more.

Games written for newer systems generally have a lot more tolerance. Since they don't have the same fixed expected cycle quota, and rely more on RTC and performance counters, and since rendering varied in time from frame to frame anyhow, they often don't care as much about hardware quirks and foibles.

Of course, there were some game systems released in an interim period. These are the Atari Jaguar, Sega Saturn, etc., and to a much lesser extent the N64 and PS1. They tend to have more CPUs than you'll see before or after this point, and very complicated hardware. Games were often written in mixtures of assembly and higher-level languages.

The Sega Saturn and Atari Jaguar are famous for their complicated hardware. In terms of the Saturn, there are

* 2 Hitachi SH-2 processors with math co-processors
* A Saturn Control Unit with DSP for geometry processing and DMA controller,
* A Hitachi SH-1 to control the CD-ROM,
* 2 VDP (Video Display Processors) to generate the picture,
* Another CPU for managing peripherals
* A Motorla 68000 to control sound, plus a sound DSP

Contrast this with the Dreamcast, which came later:
* 1 Hitachi SH-4 processor
* 1 PowerVR GPU
* 1 sound chip with integrated ARM7DI processor

It's a much cleaner, easier-to-develop-for architecture, despite sharing some roots with the Saturn. 

Despite being released within months of the Saturn, the PS1 has a much more modern architecture. The N64 is a little funky, but Nintendo put in the effort to give it a more modern development process.

Similarly, consoles released before these are simpler and more streamlined.

### What this means for accuracy
If you're developing an older-console emulator, you're likely going to focus on accuracy. Maybe to the point of cycle granularity, where each component is stepped exactly the right amount of time in synchrony with other components.

A typical emulator loop an older emulator might look like:

```python
while emulating:
    step_cpu() # process 1 CPU cycle

    step_ppu() # process 3 pixels
    step_ppu()
    step_ppu()

    step_apu() # process audio

    draw_to_screen()
```

Whereas a newer emulator doesn't care so much about that, and a typical loop may look like:

```python
while emulating:
    emulate_about_1_frames_worth_of_CPU_cycles()
    draw_to_screen()
```

That isn't to say you CAN'T or SHOULDN'T closely emulate newer hardware; projects like parallei-RDP have really made some amazing strides in emulation accuracy. It just means that games are much more tolerant of inaccuracies, and often we will make our emulators less accurate in order to increase emulation speed.
