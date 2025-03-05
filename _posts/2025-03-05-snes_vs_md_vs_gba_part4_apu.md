## Super Nintendo vs. Genesis/MegaDrive vs. GameBoy Advance, part 4: APU

I've decided to order these in order of power from least to greatest. This section is fairly clear-cut.


### A very quick primer on bit depth and sampling rate
Sound is recorded by periodically taking samples of the pressure of the air against a microphone. It is done a certain number of times per second ("sampling rate") and with a certain accuracy ("bit depth").

Due to how our ears work, you need a sampling rate of at least 40-48kHz. Anything less than this, and the sound sounds increasingly "muffled" as higher frequencies are lost. Anything more, and we don't hear the higher frequencies that are captured.

The other important factor is bit depth, or how accurate the samples are. If they aren't very accurate, the playback will sound more "noisy," with a white noise-like hiss.

CD-quality sound is 44,100Hz with 16-bit samples. MOST PEOPLE can't hear an increase in bit depth or sampling rate. However, when PRODUCING the audio, it can be useful to do so at higher bit depths and sampling rates, so as not to lose data.

### The specs
* GBA
  * Overall sound is mixed and output either 9-bit @ 32,768 Hz, 8-bit @ 65,536 Hz, 7-bit @ 131,072Hz, or 6-bit at 262,144Hz.
  * Hardware resources:
    * 2 square-wave channels like the GameBoy Color
    * A "wave" channel, which is an upgraded version of the GameBoy Color's, capable of outputting very basic samples or, with difficulty, an audio stream
    * A "noise" channel, which generates things like snare hits or cymbal sounds or whooshing noises
    * 2 8-bit digital audio channels, for stereo output or mono mixing. These can take samples at almost any rate you want to program, using one of the 4 hardware timers for each
    * This totals 6 channels of audio
    * Software can mix many more channels to go into the 8-bit outputs
  * All audio control must be done by the main CPU, which is also busy doing other things
  * Playback for the digital audio channels briefly interrupts the CPU to DMA new data in thousands of times per second
* Genesis
  * Overall, sound output here is a little funky. Each channel is output 12-bit @ 53,267 Hz in turn, and there are 6 channels. This is kind of like mixing it, but is cheaper on the hardware. It also leads to a bit of distortion on older models. The audio output thus changes at 6 times that rate, or 319,602 Hz. Each channel is output one after another in turn.
  * Hardware resources:
    * A dedicated CPU to manage audio, the Z80, with 8KB of RAM
    * Said dedicated CPU can access the rest of the system by interrupting the main CPU
    * The music chips can also be driven by the main CPU if you want
    * A Texas Instruments SN76489 (legacy from the SMS):
      * 3 square-wave channels
      * 1 noise channel
    * A Yamaha YM2612:
      * 6 4-operator FM synthesis channels with various algorithms and configurations
      * ADSR envelopes
      * 1 low-frequency oscillator
      * 1 of the FM channels can instead be switched into digital audio mode, allowing for 8-bit samples at up to 32kHz. This must be timed in software
* SNES
  * Overall sound is 16-bit, 32kHz - leading to a slightly muffled sound compared to the Genesis
  * Hardware resources:
    * A dedicated CPU (SPC-700) to manage audio, kind of similar to a 6502, with a few extra audio-related instructions: the SPC-700, with 64KB of RAM
    * This dedicated CPU *must* be used to manage audio, all communications with the audio synthesis hardware must come from here
    * This dedicated CPU has no direct access to the rest of the system: the CPU handles it all
    * S-DSP managed by the CPU:
      * 8 16-bit digital audio channels
      * ADSR envelopes
      * Built-in stereo panning and noise waveform generation
      * Built-in echo effect
      * Built-in pitch modulation

### Comments

So, if you want multi-channel GBA audio, you have to do it in software. You're limited to 8-bit 32kHz, which is half the bit depth of the SNES, though you do get stereo, and can simulate 9-bit if you use both channels in mono (this isn't worth it though).

So if a game wants a nice complicated sound, it not only has to manage reading the music and playing the notes, it also has to handle synthesizing the sounds all on its own. This is why super CPU-heavy games such as 3d games often didn't bother with music during gameplay, and why so many games had such poor sound. As we discussed in the CPU section, the CPU already had its hands pretty full most of the time.

It was rare for GBA games to sound good. Even the best-sounding ones like Pokemon Ruby still didn't sound GREAT.

Now, the Genesis could reproduce the highest frequencies. This is just about the one advantage it had over the SNES. Working with the FM chip wasn't easy, and most developers just used the presets Sega provided, leading to some pretty underwhelming-sounding games. Those that really used the hardware well could shine, of course, and also those that did poorly could hurt your ears. The 8-bit sound channel was of limited use, since it needed to be streamed in real-time sample by sample. You're not doing that from the M68000 if you're playing a game, and the Z80 only had 8KB of RAM to store samples. It also had to use most of its power just to play those samples, let alone playback other audio or mix it easily. There *are* a few homebrew ROMhacks of games like Sonic 1 that stream high-quality audio in real-time, but cart sizes were a huge constraint on this in reality. 

However, the SNES can do anything the Genesis could, theoretically, except reproduce frequencies above about 16kHz. You could record the output of FM channels setup however you want and use them as samples. Or you could use voices. Or you could use real instrument recordings, or whatever you wanted. The average SNES game sounded much better than the average Genesis game. Whether you prefer high-quality Genesis soundtracks or SNES ones is up to opinion, but the fact is the SNES could do what the Genesis could in audio in almost every way, plus a lot more.

### Ranking
1. SNES - 2 points (1 currently) = 3
2. Genesis - 1 point (1 currently) = 2
3. GBA - 0 points (4 currently) = 4

#### Final ranking after CPU, GPU, and APU
1. GBA - 4 points
2. SNES - 3 points
3. Genesis - 2 points

BUT REMEMBER. Rankings weren't the point of these posts. They're just in there for fun. I could easily add more categories the SNES or Genesis could score highly in, like ease of multiplayer...
