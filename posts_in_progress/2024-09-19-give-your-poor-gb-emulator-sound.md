## Give your poor GameBoy emulator some sound!

No, really! So many people get Pokemon working and then don't add audio. It's not that hard! My (fairly decent) implementation is only 539 LoC! [It can be found here](https://github.com/raddad772/jsmooch-emus/blob/main/jsmooch-lib/src/component/audio/gb_apu/gb_apu.c). It's not perfect, but it makes fairly good-sounding sound for games.

It's really very simple if you understand how it works at a high level. I'm here to help with that.

### But wait! Even getting audio output is a pain!
I used [miniaudio](https://miniaud.io). It's a simple library that gets you audio output on many different platforms. [My wrapper using it](https://github.com/raddad772/jsmooch-emus/blob/main/jsmooch-gui/src/audiowrap.cpp) is less than 200 LoC for initialization and usage. All you need to do is some minimal setup and supply it with buffers.

I have mine setup to take buffers that are 1/60th of a second, so I just generate a buffer each time I generate a frame, and I am golden.

### OK, let's talk about GameBoy sound synthesis
First, some links. [PanDocs has issues](https://gbdev.io/pandocs/Audio.html) but is a good overall source; [this link here](https://gbdev.gg8.se/wiki/articles/Gameboy_sound_hardware) is overall more accurate but less usable.

But don't get into those before we go over the high level here!

You get a total of 4 channels (2 pulse, 1 wave, and one pseudorandom noise channel - we'll get to what that means soon), and they're very very simple to work with.

There's four main components to worry about: sound synthesis (easy), IO (easy!), mixing (fairly easy), and built-in "effects" a.k.a. the Frame Counter (also easy).

IO refers to reading and writing the audio registers to set the parameters for the sounds. Sound synthesis refers to actually generating the waveforms, mixing refers to mixing them all together so you can have more than one sound, and the Frame Counter is the mechanism the gameboy uses to limit the notes and provide limited built-in effects.

### Starting simple
Seriously, start out simple. Just start with channel 1 (the first pulse channel). Implement it and get some audio out. You can add the others as you go, and getting just that output will get you a long way toward being done.

### Sound Synthesis: it really is easy

The GameBoy's sound synthesis hardware is just a bunch of counters. Literally.

To create an output sound, each tick:
1) a counter is incremented until it overflows.
2) When it overflows...
   3) it is reloaded
   4) a new sample is chosen

It's really no more complicated than that.


#### Sound Synthesis: pulse channel 1 & 2
Here is the code for that:

```c
static void tick_pulse_period(struct GB_APU *this, int cnum)
{
    struct GBSNDCHAN *chan = &this->channels[cnum];
    if (chan->on) { // If the channel is on,
        chan->period_counter++; // 1) Increment the counter
        if (chan->period_counter >= 0x7FF) { // 2) If the counter overflows...
            chan->period_counter = (i32)chan->period; // 3) Reload the counter
            chan->duty_counter = (chan->duty_counter + 1) & 7; // 4a) Increment the duty cycle tracker (where in the waveform we are). Range 0-7
            chan->polarity = sq_duty[chan->wave_duty][chan->duty_counter]; // 4b) Set our channel's output to a value 0...1
        }
    }
}
```

And the values are picked from this:
```c
static i32 sq_duty[4][8] = {
        { 0, 0, 0, 0, 0, 0, 0, 1 }, // 00 - 12.5%
        { 1, 0, 0, 0, 0, 0, 0, 1 }, // 01 - 25%
        { 1, 0, 0, 0, 0, 1, 1, 1 }, // 10 - 50%
        { 0, 1, 1, 1, 1, 1, 1, 0 }, // 11 - 75%
};
```
There's also code to setup those parameters, but actually synthesizing the sound couldn't be easier.

#### Sound Synthesis: what about the wave channel?
This looks so similar! Take a look:

```c
static void tick_wave_period_twice(struct GB_APU *this) { // this is ticked twice at a time because of the way I handle audio ticks in my emulator: they are sent at the same rate as the CPU, so 1MHz
    struct GBSNDCHAN *chan = &this->channels[2];
    for (u32 i = 0; i < 2; i++) {
        if (chan->on) { // If channel on...
            chan->period_counter++; // 1) increment the counter
            if (chan->period_counter >= 0x7FF) { // 2) if it overflows...
                chan->period_counter = (i32) chan->period; // 3) reload the counter
                chan->duty_counter = (chan->duty_counter + 1) & 31; // 4a) Increment the duty cycle tracker (where in the waveform we are). Range 0-31
                //chan->period = chan->next_period;
                if ((chan->duty_counter & 1) == 0) { // 4b) We load samples from waveRAM to internal memory every other cycle, because 1 byte = 2 samples
                    u8 p = chan->samples[chan->duty_counter >> 1];
                    chan->sample_buffer[0] = p >> 4;
                    chan->sample_buffer[1] = p & 15;
                }
                chan->polarity = chan->sample_buffer[chan->duty_counter & 1] >> chan->env.rshift; // 4c) We set our output to the value of the sample we loaded, and add a little volume reduction
            }
        }
    }
}
```

#### Sound synthesis: the noise channel

The noise channel differs in that it counts down instead of up, and of course the particulars of how it handles bits.

It's based around an LFSR or Linear Feedback Shift Register. These are a simple and cheap way to make random-seeming (but NOT random!) numbers with very few transistors. Basically they work by taking some shift register (a register that shifts things out 1 bit at a time), doing a little math to it, and getting bits out of it.
Check out the code:

```c
static void tick_noise_period(struct GB_APU *this)
{
    struct GBSNDCHAN *chan = &this->channels[3];
    if (chan->on && (chan->clock_shift < 14) && (chan->period != 0)) { // If the channel is on
        chan->period_counter--; // 1) Decrement the counter
        if (chan->period_counter <= 0) { // 2) If we hit 0...
            chan->period_counter = (i32)chan->period; // 3) Reload the counter
            
            // Here, we go into the particulars of how an LFSR (Linear Feedback Shift Register) works. Don't worry too much about this yet
            u32 flipbits = ~(chan->short_mode ? 0x4040 : 0x4000);
            u32 lfsr = chan->duty_counter;
            u32 l2b = lfsr;
            lfsr >>= 1;
            l2b = (l2b ^ lfsr) & 1;
            lfsr &= flipbits;
            lfsr |= l2b * ~flipbits;
            chan->duty_counter = lfsr;
            
            // 4) we choose the output value just based on the opposite of the lowest bit of the LFSR
            chan->polarity = (chan->duty_counter & 1) ^ 1;
        }
    }
}
```

That is all the code needed to get fairly in-depth gameboy sound synthesis! That's barely anything!

### Sound synthesys and timing
This only works, of course, if you send out ticks at the correct time. Thankfully, that's easy!

Each m-cycle tick of my CPU (outside of turbo mode) I call the following function (so at 1MHz):

```c
void GB_APU_cycle(struct GB_APU* this)
{
    tick_pulse_period(this, 0);
    tick_pulse_period(this, 1);
    tick_wave_period_twice(this);
    tick_noise_period(this);
    // now do some frame sequencer stuff if it's time
```

### Let's do a little IO...
The registers themselves can be found in the above-linked reference documents, 
