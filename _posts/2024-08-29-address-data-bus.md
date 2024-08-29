## The Address and Data Busses: How to Understand Mappers

So, you're working on a gameboy or NES emulator, and mappers are throwing you for a loop. Why are banks like that? Why do I have to write to it 1 bit at a time? Why does it do these funky things to memory?

And if you're not that far...How does a CPU writing to a memory address tell the PPU to do something?

### Basics of signaling
A typical CPU has a bunch of different pins, but usually there are some signals similar to "data strobe." This signal is used by many CPUs to indicate that the data on the data bus (we'll explain what that is later) is valid and should be acted upon, or that it is requesting data.

The 6502 is a bit of a special case in that every single cycle is a read or write. The 65c02 fixed this by adding a memory request pin.

Similarly, most CPUs have a R/W pin, where logical 0 indicates read and 1 indicates write, or vice versa.

To get these signals from component to component, they must be output through pins, then travel along the board in the traces (the silver or golden lines you see all over a PC board), then travel into a component through more pins.

A set of traces shared by multiple components is known as a "bus."

These busses are shared by multiple components.

### Chips in reality
The first thing we have to remember, is that electronic components cost money and space. Chips like processors interact with other chips through pins which go to traces on a board, and every single signal must be routed properly, somehow. Modern PCBs have multi-level construction, allowing traces to weave over and under each other, but this was not always a thing.

The pins on a chip are limited, too. There are various standardized packages that can be manufactured, and a custom package is extraordinarily expensive.

Both the 6502 and the Z80 came in 40-pin packages. This means at most 40 signals in and out to the world. They need clock signals, status signals, address and bus signals, and more.

### Chips in practice: who wants what?

Most processors of the era could address 64kb of RAM. That's 2 to the 16th power, or a 16-bit "address bus." That meant that it required 16 traces to fully resolve an address, and 8 more for the 8-bit data bus, for a total of 24 traces routed around the board.

### Who can talk? Decoding addresses

Like we said above, multiple components share the same buses. Let's use the NES as an example. The CPU communicates with...

1) RAM,
2) ROM (the games) and mappers,
3) The APU (although this happens internally to the Ricoh 2a03, which has an m6502 embedded along with basic OAM DMA logic and I/O logic)
4) The PPU

So how does each one know if the CPU is trying to talk to it? How does it know it is being addressed by the CPU?
 
Each component has a pin, often called "CHIPSEL" for "chip select," that is tied to an "address decoder." An address decoder partially or fully decodes the address bus. This is done in a pretty simple way.

Let's pretend we have a super-simple 6-bit address bus. Our address space of 6 bits allows addresses from 0-63. We have 3 components arranged as so:

```
 Addr  Component
-----  ---------------------
 0-15  16 bytes of RAM
   16  PPU
32-47  16 bytes of game ROM
   48  APU 
```

Each component will need to decode that address, and use the result to their CHIPSEL. So let's take the address, 16, for the PPU. In binary, that looks like

010000

Bit 4 is set and bits 0-3 and 5 are 0. A basic, full address decoder will first invert the bits that are supposed to be 0, yielding...

``` 
 1       bit left how it is
1 1111   inverted bits
```

Then and the result together. The result of 6 1's ANDed together, is 1, which will notify our PPU that we are trying to talk to it. The result of that address decoding will be fed to the CHIPSEL pin of the PPU.

Alternately, the address lines can just be passed into the PPU, and this decoding can happen internally, with the same effect.

### Again, electronics aren't free
To do this decoding, I need to use electronic components. Again, these are not free in cost or space on the board or chip. Each address and data signal it needs is another pin, as well as internal logic.

How can we save some complexity?

Well, if you recall, our memory map has a lot of gaps:

```
 Addr  Component
-----  ---------------------
 0-15  16 bytes of RAM
   16  PPU
32-47  16 bytes of game ROM
   48  APU 
```

Currently, we're decoding all 6 bits of the address. But any value in the range of 16-31 would really be fine, wouldn't it? After all, nothing else would respond. So why are we wasting extra NOT and AND gates?

Instead of decoding all these bits...

543210

We could just worry about these:

54xxxx

So we take 5 and invert it with a NOT gate, then AND that with bit 4, for our CHIPSEL. We've saved out on 83% of the gates we need! Now any address from 16-31 will be decoded as pointing at the PPU.

Our memory map will now look like this:

```
Addr  Component
-----  ---------------------
 0-15  16 bytes of RAM
16-31  all addresses = PPU
32-47  16 bytes of game ROM
   48  APU
```

### Using it to our advantage
A partial decoding like this has other advantages. For instance, it allows one decoder to decode all of the RAM addresses, instead of needing a decoder for every byte!

So in the case of a decoder for the RAM chip, we would want to do...

54xxxx

NOT 5, NOT 4, and AND the result together, for our CHIPSEL. Now any address where both bits 5 and 4 are 0 will trigger the RAM!

We can then pass just the 4 lowest bits of the address into the RAM chip itself. After all, it's only 16 bytes big, it only needs 4 address lines. Reads and writes to RAM are now happening, and the address decoders in front of the other components assure they won't try to service the read or write request.

Any bit we don't decode is effectively ignored, and can be in any state.

### Mirror, mirror
In an NES, there is 2KB of RAM accessible from $0000-$07FF, but there are "mirrors" of this RAM from $0800-1FFF. So uh...why? Why, when the CPU accesses $1100, is it the same as accessing $0100?

Let's look at the bits in these addresses:

```
$0100 - $100 bytes into RAM - 0000 0001 0000 0000
$1100 - $100 bytes into RAM - 0001 0001 0000 0000
$2100 - doesn't go to RAM   - 0010 0001 0000 0000   
```

We can see here, that the address is partially decoded. The bits involved in the 0...7FF portion (bits 0...10) are passed through to the RAM. But in order to ensure that RAM is only accessible from $0000-07FF, the address decoded would need to decode ALL the 5 other bits.

However, WHY should we do this? We can just decode the top 3 bits, send the bottom 10 into the RAM, and ignore the rest. This will assure that the RAM is selected properly for $0000-07FF, as well as "mirroring" it from there, since it only gets 11 bits of the 13 bits of address that can trigger the decoder.

So in a real NES, decoders are used like this:

```
How many bits         Value decoded         What is there   Addresses     Used bits for component        Ignored bits/num mirrors
3 most significant    000. .... .... ....   RAM CHIPSEL     $0000-$1FFF   11 to select address in RAM    2/4
3 most significant    001. .... .... ....   PPU CHIPSEL     $2000-$3FFF   3 to select register           10/1024
11 most significant*  0100 0000 000x xxxx   APU CHIPSEL     $4000-$401F   5 to select register           0/0

*This one is internal to the Ricoh 2a03. I do not know what is on the external address bus here. I do know that if something responds at $4000-401F externally it will just be ignored.
```

Those aren't the only components in the NES, however. There's also, the game cartridge...and...the PPU bus!

### The PPU bus?
The PPU has an entirely different address space. $0000-1FFFF is the CHR tables, mapped by the cart. $2000-2FFF is 8KB of VRAM for the name and attribute tables, and $3000-3FFF is mappable, again, by the cart. This is a total of 14 bits of address space, with its own separate 8 bit data bus as well.

### A typical cartridge
A North American NES uses a 72-pin cartridge connector. 16 of these are used for the CPU address bus, 8 for CPU data bus, 1 for CPU RW, 14 for PPu address bus, 8 for PPU data bus, and another 1 for PPU RW. There are lots of other pins for other purposes, but that already sets us up with 45 of the 72 pins just used for signalling with the CPU and PPU!

Let's take Super Mario Bros.' layout. NESCartDB says it has a 32KB game ROM, and 8KB of CHR ROM. As we know, the relevant CPU/PPU memory maps look like this:

```
CPU
$0000-7FFF: nothing here (as far as the cart cares) 
$8000-FFFF: 32KB game ROM

PPU
$0000-1FFF: 8KB CHR ROM
$2000-3FFF: nothing here (as far as the cart cares)
```

We can see here exactly what we need. To decode the address for the game ROM CHIPSEL, we need only 1 bit, the topmost bit. If that bit is set, the ROM will respond with whatever is in it at the address specified by the low 15 bits.

The CHR ROM CHIPSEL is similar, except it checks that bits 15 and 14 are 0. If they are, its CHR ROM will respond.

### PPU mirroring
One more detail that's interesting here, is the PPU mirroring. Inside .nes files is a bit flag that indicates 1-screen, horizontal, vertical, or 4-way mirroring.

Recall the PPU screen setup. There are 4 "nametables," each $400 bytes, starting from $2000 in this order:

```
1 2
3 4
```

But the PPU itself only has $800 bytes of RAM. So how is mirroring accomplished?

Well, the cartridge actually passes the PPU address and data lines back into the console. Horizontal and vertical mirroring are the only ones allowed by regular mapperless games, and it is accomplished a little like this.

```
PPU read nametable 1-  $2000 0010 0000 0000 0000
PPU read nametable 2 - $2400 0010 0100 0000 0000
PPU read nametable 3 - $2800 0010 1000 0000 0000
PPU read nametable 4 - $2C00 0010 1100 0000 0000
                                  ^^
                                  These two bits actually select nametable 0-3 as long as we're >= $2000
                                  These bits are A10 and A11 meaning Address Bit 10 and Address Bit 11
```

Vertical mirroring, where the $800 bytes of PPU RAM are mapped to $2000-$27FF, leaving $2800-2FFF to mirror $2000-27FF, is accomplished by forcing A11 to 0. That way even if the PPU tries to read from $2800, the signal passed to the PPU RAM will still be $2000.

Horizontal mirroring, where the $800 bytes of PPU RAM are logically split in half and mapped at $2000-23FF and $2800-2BFF, leaving $2400-27FF as a mirror of $2000-23FF, and $2C00-2FFF as a mirror of $2800-2BFF, is accomplished by swapping A11 and A10 and then making the new A11 be 0. The traces on the cartridge will literally go from A11 to A10.

### But what if we want more storage?
What if we want more storage than 32K game ROM and 8K CHR ROM? Or on the GB, more than 32K game ROM? Or on a ZX Spectrum, more than 48K of RAM, etc.?

Well, we use banking!

Banking is when we say, LOGICALLY, this area of ROM/RAM will ACTUALLY access this OTHER area of ROM/RAM, and it's accomplished in a very simple manner in early chips.

Let's take a look at Castlevania, which used UxROM boards (iNES mapper 002). Instead of CHR ROM, it used 8KB of CHR RAM, which the CPU would decompress tiles to. It also had 128KB of ROM.

Looking at the memory map the NES sees when this cart is inserted, we get:

```
0x8000-0xBFFF: selectable 16KB bank
0xC000-0xFFFF: last 16KB of the cartridge

write to any address in 0x8000-0xFFFF to set the bank
```

How does that work? It turns out it's pretty simple.

There's 2 components that are addressed on the board by the CPU (we're ignoring the PPU for now): the ROM, and the bank controller.

We have the following signals to work with:

* 16 address lines, A0-A15
* 8 data lines, D0-D7
* R/W line

A15 selects the cartridge in general, and if A15 *AND* WRITE, a new value will be latched into the 74-161 counter IC from D0-D3.

Having 128KB of ROM, that requires 17 lines of address data to address it all. The 16th line (A15) is used to select the cartridge at all, leaving 15 lines free for addressing ROM. Since A14 selects between "lower (changing)" and "upper (fixed)" banks, that means we have 14 bits free (16KB) for selecting which byte in ROM. Since we need 17 bits to fully address it, that means we need to get 3 bits from somewhere.

A 17-bit address into the ROM looks like this:

```
AAABBBBBBBBBBBBBB
3 A's followed by 14 B's.

The signal we get from the CPU looks like this

CDBBBBBBBBBBBBBB

where C is cart select, D is bank select, and B are bits that are passed through to the ROM.
```

So where do those upper 3 bits come from? The answer to that is simple: the output of the counter IC supplies them. It takes in a value and just repeats it forever when it is selected (there is a "lower bank" address, A14=0). It provides the upper 3 bits for the ROM address.

When A14=1, it uses a value of 111 for the upper 3 bits, so that it's always in the last 16KB of the ROM.

A0-A13 from the CPU are combined with either that IC output or 111 to generate a full address.

### More complicated mappers
There are other more-complicated mappers, but they all work the same way: they have a programmable way to provide the upper bits of the address to the ROM. CHR ROM banking works the same: the upper bits will select any bank(s) and the rest will determine where inside that bank specifically will be addressed.

There are many different ways of loading those bits, such as MMC1's serial-to-parallel shift register, which only looks at D0 and shifts bits in 1 at a time, outputting the full value when enough bits are shifted.

And in many places, addresses are only partially decoded, leading to the ability to access the same register at many different addresses, simply because it's cheaper and easier to do it that way from a hardware perspective.

### Conclusion
I hope this helps you understand banking, mappers, etc. 