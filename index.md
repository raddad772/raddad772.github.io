# Emulators in pure JavaScript!
I'm writing emulators for complex systems. in pure JavaScript - no WebAssembly hacks...at least until we start with JIT code.

Why? Because they said it couldn't be done. Because I'm literally crazy. Because I can.

I don't personally like JavaScript as a language, but there's already a billion emulators in C.

I'll also be posting about topics of interest to me here, on this blog. Like an introduction to the 65816, or a story about optimizing some assembly code, or notes on getting JavaScript to perform well.

## Emulators and progress
1. Systems
  1. SNES - 65816 CPU emulation is partially working (see below)
  2. Weird test system - a simple computer with 1MB of RAM, a WDC65c816 CPU, and a 256x256 monochrome display. Going to be used to make a counterpart to Easy6502, the great page that lets you learn 6502 assembly in your browser
1. CPUs
  1. WDC 65c816
    * A 16-bit extension to the 6502 that was in everything, from Commodore64 to Apple II to the NES. If it didn't have one of these, it had a Z80. Currently it is running around 2-4MHz on my PC, which is good, since I haven't put effort into optimizing it.

## What else?
I've collected some resources that I have found helpful. That's about it.
