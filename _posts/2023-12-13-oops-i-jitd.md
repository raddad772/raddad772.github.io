## Oops, I JIT'd

I hear all the time, that JIT is a confusing and challenging idea for many emulator developers. Really, it couldn't be further from the truth!

In this post, I'll go over the theoreticals, and in future posts, we can build a functioning R3000 JIT together! 

### R3000?
That's the MIPS R3000, the CPU inside the Sony PlayStation. All instructions are 32-bit words. It has an annoying pipeline and a few caveats, but it's pretty straightforward to write a JIT for.


### Steps

It's a completely good idea to start out planning a JIT from scratch. But to show how easy it is, we're going to start with a functional interpreter and go from there.

1) Write an instruction decoder
2) Write an interpreter
3) Modify this interpreter to become a cached interpreter
4) Modify the cached interpreter to use native code instead
5) OOPS! I JIT'd!

For the sake of my sanity, I will assume you can already handle steps #1 and #2. This is not an intro to the basics of emulation.

### Step 3: cached interpreter

#### The problem that is solved
What is a cached interpreter?

Basically, when emulating instructions, there is overhead in decoding them. Take a look at this R3000 interpreter loop:

The code snippets here are in AssemblyScript, a very close cousin of TypeScript meant to compile into WebAssembly. 

```typescript
// Trigger an exception if needed
if (this.pins.IRQ && (this.regs.COP0[12] & 0x400) && (this.regs.COP0[12] & 1)) {
    this.exception(0, this.pipe.get_next().new_PC !== 0);
}

// If we don't have anything in the pipeline (which can happen for various reasons), fetch and decode a new item
if (this.pipe.num_items < 1)
    this.fetch_and_decode();
let current = this.pipe.move_forward();

// Execute the decoded function
current.op.func(current.opcode, current.op, this);

// Move the delay slots through
this.delay_slots(current);

// Clear current pipeline entry
current.clear();

// Fetch and decode after execution
this.fetch_and_decode();

cycles_left -= 2;
```

For every 2 cycles of execution, at least one fetch_and_decode() must happen. Let's take a look inside there...

```typescript
fetch_and_decode(): void {
    let IR: u32 = this.mem.CPU_read(this.regs.PC, MT.u32, 0);
    let current = this.pipe.push();
    this.decode(IR, current);
    current.opcode = IR;
    current.addr = this.regs.PC;
    this.regs.PC += 4;
}

decode(IR: u32, current: R3000_pipeline_item_t): void {
    let p1 = (IR & 0xFC000000) >> 26;
    let p2 = (IR & 0x3F);
    if (p1 === 0) {
        current.op = this.op_table[0x3F + p2]
    }
    else {
        current.op = this.op_table[p1]
    }
}
```

A BUNCH of function calls happen, along with bit shifting to decode the instruction, and this is an extremely simple decoder.

Once we have the instruction, yet more decoding must be done. 

```typescript
export function R3000_fXORI(opcode: u32, op: R3000_opcode, core: R3000): void {
    let rs = (opcode >>> 21) & 0x1F; // Decoding
    let rt = (opcode >>> 16) & 0x1F; // More decoding
    let imm16: u32 = opcode & 0xFFFF; // More decoding!

    R3000_fs_reg_write(core, rt, imm16 ^ core.regs.R[rs]); // The actual work of the instruction is done here
}
```

That's multiple function calls, multiple bit shifts, and more, done EACH TIME this instruction is executed. And this looks much worse for more complex decoding schemes.

Surely there must be some way we can cache this information so we don't have to do it over and over and over, yes?

#### The problem is solved: the block cache
Indeed we can! We're going to be moving into more abstract examples now. Where code is included, it is frequently not syntactically perfect and is closer to pseudocode.

Let's say we have a program that starts in memory at 0x100. We can take that instruction, and translate all the possible info out of it.

Then, in some theoretical structure, we could write:
```
Pointer to function, arg1, arg2, arg3
```

If we kept track of where that structure pointed to, we could even put multiple in a row!

So then we would have a structure that says
```
start: 0x100
data:
Pointer to fXOR, arg1, arg2, arg3  // 0x100
Pointer to fLWL, arg1, arg2, arg3  // 0x104
Pointer to fJR, arg1, arg2, arg3  // 0x108
```

Then, if the PC jumps to 0x100, we can find this structure, and instead of going through all the work of decoding it, we can just call the provided function with the provided operands. If we've bothered to do a full decoding for the instruction first, then we've saved a LOT of overhead on each instruction! We can go further, but that's far enough for now. 

#### Block problems

It's more complicated than that, of course, because there are lots of details to managing these blocks. Here's a brief discussion on the topic in a Q&A format

##### Q: How do I tell which memory I have a cache of?
 There's a lot of ways to handle this! The basics is that you want to translate BLOCKS OF INSTRUCTIONS at a time. You can then mark it in a big bit field, or a hash table, or however you'd like.
 
Make note that there is a nasty tendency for code to overwrite itself; on some platforms, you need to keep track of if writes overwrite areas of memory you've previously cached.

##### Q: How do I determine how big to make a block?
You set an arbitrary maximum block size, and you stop compiling at unconditional jumps, and some conditional ones.

It's a big optimization to structure your flow such that conditional jumps that jump back into the current block can be left as-is.

##### Q: Which block do I start with?
A very simple way to do this is to modify the code above. Instead of 

```typescript
let current = this.pipe.move_forward();

// Execute the decoded function
current.op.func(current.opcode, current.op, this);

// Move the delay slots through
this.delay_slots(current);

// Clear current pipeline entry
current.clear();

// Fetch and decode after execution
this.fetch_and_decode();
```

we can do...

```typescript
let current = this.pipe.move_forward();
let current_op = this.block_cache.get(current.PC);
current_op.func(current_op.arg1, current_op.arg2, current_op.arg3);

current.clear();

// closer to pseudocode here
function get(PC) {
    if (already_translated[PC]) return cached_instruction[PC];
    
    cached_instruction[PC] = translate_instruction(PC);
    already_translated[PC] = true;
    return cached_instruction[PC];
}
```

You see here that we just query if we have a cached instruction there and return it if so. This query is faster than a decode.

This does not respect blocks, though, because the fundamental nature of your emulator changes as you move to cached interpreter.

#### The fundamental change to the interpreter loop
Previously, we had been going one instruction/2 cycles per loop, because that is how the R3000 operates (IIRC). However, cached interpreted code and JIT code share in that they are fundamentally less accurate (unless you put in a lot of effort: you CAN do cycle-accurate JIT still).

There's tradeoffs here, but most JIT is done for speed over accuracy, so we'll assume we want MAXIMUM SPEED.

To incorporate this, instead of interpreting instructions one at a time, we will interpret one block at a time.

Old loop looked like:

```typescript
while (cycles_left > 0) {
    do_cycle();
    cycles_left-=2;
}
```

And the new loop will look like:

```typescript
while (cycles_left > 0) {
    block = get_next_block(core);
    block.execute_whole_block();
    cycles_left -= block.cycles;
}

function execute_whole_block() {
    while (instruction in self.instructions) {
        instruction.execute();
    }
}

function get_next_block(core: R3000) {
    // First, check if block is already created for the current PC
    if (already_translated[PC]) return cached_block[PC];
    
    // If it is not yet created, create it!
    cached_block[PC] = translate_block_of_instructions(PC);
    already_translated[PC] = true;
    return cached_block[PC];
    
    // Return the block
}

function translate_block_of_instructions(PC) {
    let block = [];
    while (block.size < MAX_BLOCK_SIZE) {
        instruction = decode_for_cached_interpreter(PC);
        block.append(instruction)
        if (instruction_should_terminate_block(PC)) break;
        PC += 4;
    }
    block.cycles = block.size * 2;
    return block;
}

function instruction_should_terminate_block(instruction) {
    if (instruction is a control flow instruction of some kind) return true;
    return false;
}
```

You can see here that you can overrun the number of cycles of the CPU you'd like to emulate pretty easily, and on platforms with variable-length instructions, the # of cycles you're executing becomes more like a guess. This is the price of SPEED!

### Step 4: from cached interpreter to JIT
Now that we've got our nice cached interpretr going, going to JIT is actually pretty easy. There are some platform-dependent details I won't go into, and there's a whole lot of JIT design we can talk in another blog post, but this should get you started.

Now that you're going and executing whole blocks at a time, think about our data structure that contains the blocks.

```
start: 0x100
data:
Pointer to fXOR, arg1, arg2, arg3  // 0x100
Pointer to fLWL, arg1, arg2, arg3  // 0x104
Pointer to fJR, arg1, arg2, arg3  // 0x108
```

We're working conceptually here. Assuming we could generate code in javascript to run like this, the structure could be changed to this:

```
start: 0x100
data:
XOR(arg1, arg2, arg3)
LWL(arg1, arg2, arg3)
JR(arg1, arg2, arg3)
```

If it is native code, we could ALMOST execute it directly now!

#### How to generate native code
The easiest way is with function templates.

Here's what fXOR looks like:
```typescript
function cached_R3000_fXOR(core: R3000, arg1: u32, arg2: u32, arg3: u32) {
    R3000_fs_reg_write(core, arg3, core.regs.R[arg1] ^ core.regs.R[arg2]);
}
```

R3000_fs_reg_write simply makes sure the register is properly updated in accordance with the rules of the internal pipeline of the R3000.

In order to do what it needs to, this function needs to:

1) Accept 4 arguments
2) Access the registers out of the "core" structure (which is a 4th argument)
3) Call a function to update the register properly according to internal pipeline rules

We can create a complementary function whose purpose is to output amd64 assembly that accomplishes the same thing. 

```typescript
function JIT_R3000_fXOR_amd64(arg1: u32, arg2: u32, arg3: u32, output_buffer) {
    // core is an assumed value, we'll get to that when we cover thunks
    x64_R3000_reg_lookup(core, arg1, 1, output_buffer);
    x64_R3000_reg_lookup(core, arg2, 2, output_buffer);
    output_buffer += 'XOR opcodes'; // put opcodes here to actually perform the XOR and put result into r3
    x64_R3000_interpreter_call(R3000_fs_reg_write, core, arg3, 3)
}

function x64_R3000_reg_lookup(core: R3000, src_reg: u32, dest_reg: u32, output_buffer) {
    output_buffer += 'x86-64 opcodes to extract the register src_reg from the R3000 structure pointed to by core, and place in x86-64 tegister dest_reg'
}

// This next function I won't go into all the details of
function x64_R3000_interpreter_call(func, core: R3000, args...) {
    
}

```

I didn't use actual assembly there, but the end output of the function should be pretty clear: the output_buffer now contains x86-64 opcodes that will execute an fXOR directly.

If we make a JIT_ version of each R3000 instruction, then we can modify the way our blocks work a little.

To compile them,
```typescript
function translate_block_of_instructions(PC) {
    let block = [];
    while (block.size < MAX_BLOCK_SIZE) {
        instructions = translate_to_x64_code(PC);      // Used to be decode_for_cached_interpreter(PC);
        block.append(instructions);
        if (instruction_should_terminate_block(PC)) break;
        PC += 4;
    }
    block.cycles = block.size * 2;
    return block;
}

function translate_to_x64_code(PC) {
    let output = [];
    let opcode = read(PC)
    if (instruction_kind(opcode) === 'XOR')
        JIT_R3000_fXOR_amd64((opcode >>> 21) & 0x1F, (opcode >>> 16) & 0x1F, rd = (opcode >>> 11) & 0x1F, output);
    else //... more instruction decodes here

}
```

When we're executing our emulator, we change things from here (cached interpreter):

```typescript
while (cycles_left > 0) {
    block = get_next_block(core);
    block.execute_whole_block();
    cycles_left -= block.cycles;
}

function execute_whole_block() {
    while (instruction in self.instructions) {
        instruction.execute();
    }
}
```

Instead of executing each instruction one by one, we can change it to jump into the instructions.

```typescript
while (cycles_left > 0) {
    block = get_next_block(core);
    jump_into_block(block); // used to be block.execute_whole_block();
    cycles_left -= block.cycles;
}
```

And we're ALLLLLLLLMOST there! We're compiling native code blocks that should work, and we still have a very similar structure to our cached interpreter! But if we try to run this, it won't work.

#### CPU resources and context switching
We're going to pivot over to C++ now. Using a very simple program (and godbolt.org), we can see what happens when we call functions.

```c++
int do_something(int b) {
    int c = b + 4;
    return c;
}
int main() {
    int a = 1;
    a += 2;
    a = do_something(a);
}
```

Annotated with diassembly, this looks like:

```c++
int do_something(int b) {
   /* asm to enter the function.        
        push    rbp                        Save rbp to stack
        mov     rbp, rsp                   Set rbp to stack pointer
        mov     DWORD PTR [rbp-20], edi    Set next spare entry on stack to edi, which is what we saved the value of a to in main()
        */

   /* asm to do next line
        mov     eax, DWORD PTR [rbp-20]    Copy the passed-in variable to register eax
        add     eax, 4                     Add 4 to the register eax
        mov     DWORD PTR [rbp-4], eax     Copy the value in eax into a stack variable
        */
    int c = b + 4;
        
    /* asm to do next line
        mov     eax, DWORD PTR [rbp-4]     Copy the value we calculated above into the register eax, which by convention is the return value register
    */
    return c;

   /* asm to end function
        pop     rbp                         Restore rbp to how it was, before we modified it for our own use
        ret                                 Return to caller
   */
}

int main() {
    int a = 1;
    a += 2;
    a = do_something(a);
}

```

So what's going on here?

Processors only have so many fast-access registers. Because of this, if you want to use one of them in code that has been called, you will find that it was probably already in use! To get around this, we see that do_something() pushes rbp to stack, and pops it before return, so that main() gets it back the same way as when it called.

This is part of the calling convention, which varies by language, compiler, and platform. Another thing we see here is that eax is NOT pushed, because the caller assumes it will be used, and then use it as the return value. The caller already cleaned up eax if it wanted to.

After a call, you basically need to give back the processor as you first got it, and follow other conventions. You can be passed parameters, etc.

If you're REALLY into optimizing your JIT, later on you'll also start doing more optimizations like condition code opitmizations, inter-block optimizations, and more, but for now, we're going easy.

So we need to keep track of which registers were modified, and save them before we get to our native code, and load them before executing a ret.

FURTHERMORE, we have some context that we know every instruction is going to need, like the address of the R3000 structure, so that opcodes can change its state. We can decide now that some register is always going to hold that address. Let's say, rbp, for now.

We want to do this automatically and seamlessly since we don't want to write it a million times.

```typescript
class block_context_manager {
    constructor() {
        this.output_buffer = [];
        this.registers_used = [];
        this.num_instructions = 0;
    }
    
    use_register(reg) {
        if (!(reg in this.registers_used)) registers_used.append(reg);
    }
    
    add_to_output(what) {
        this.output_buffer.append(what);
    }
    
    finish() {
        let real_output = [];
        
        // Translate parameters we've gotten. This will be jumped to as if it has one parameter, core: R3000 .
        // As if it were a function that looked like
        //  execute(core: R3000): void {}
        // First, make sure we didn't already use a reserved register
        if ('rbp' in this.registers_used) {
            throw('rbp is reserved for core structure')
        }
        // Save rbp and load pointer to "core" from stack
        real_output += 'push rbp, [sbp-4]'
        real_output += 'mov rbp, [sbp-4]'
        
        // Finish context switch in by saving used registers
        for (reg in this.registers_used) {
            real_output += 'push ' + reg; // Except the opcodes for it, not a string
        }
        
        // Add in the generated opcodes
        real_output += this.output_buffer;
        
        // Start context switch out by reloading those registers
        for (reg in this.registers_used.reverse()) {
            real_output += 'pop ' + reg; // Except the opcodes for it, not a string!
        }
        
        // Restore the old rbp. The first thing we did was push it
        real_output += 'pop rbp'
        
        // Return to caller when we're done
        real_output += 'ret'
        
        return real_output;
    }
}

class block {
    constructor() {
        this.num_cycles = 0;
        this.address = 0;
        this.instructions = [];
    }
}
```

And now, while we're compiling a block, this

```typescript
function translate_block_of_instructions(PC) {
    let block = [];
    while (block.size < MAX_BLOCK_SIZE) {
        instructions = translate_to_x64_code(PC);
        block.append(instructions)
        if (instruction_should_terminate_block(PC)) break;
        PC += 4;
    }
    block.cycles = block.size * 2;
    return block;
}
```

Becomes this:

```typescript
function JIT_block_of_instructions(PC) {
    let bcm = block_context_manager();    // used to be, block = []
    while (bcm.num_instructions < MAX_BLOCK_SIZE) {
        instructions = translate_to_x64_code(PC, block_context_manager);
        bcm.add_to_output(instructions)  // used to be block.append(instructions)
        if (instruction_should_terminate_block(PC)) break;
        PC += 4;
    }
    // Now make our block structure to return
    block = compiled_block();
    
    block.instructions = bcm.finish(); 
    
    block.cycles = bcm.num_instructions * 2;
    
    return block; 
}
```

And don't forget to modify our fXOR example output function and helpers so they report their register usage!

```typescript
function JIT_R3000_fXOR_amd64(arg1: u32, arg2: u32, arg3: u32, output_buffer, block_context_manager) {
    // core is an assumed value, we'll get to that when we cover thunks
    x64_R3000_reg_lookup(core, arg1, 1, output_buffer, block_context_manager);
    block_context_manager.use_register('r1')
    x64_R3000_reg_lookup(core, arg2, 2, output_buffer, block_context_manager);
    block_context_manager.use_register('r2')
    output_buffer += 'XOR opcodes'; // put opcodes here to actually perform the XOR and put result into r3
    block_context_manager.use_register('r3')
    x64_R3000_interpreter_call(R3000_fs_reg_write, core, arg3, 3, block_context_manager)
}

function x64_R3000_reg_lookup(core: R3000, src_reg: u32, dest_reg: u32, output_buffer, block_context_manager) {
    output_buffer += 'x86-64 opcodes to extract the register src_reg from the R3000 structure pointed to by core, and place in x86-64 tegister dest_reg'
    block_context_manager.use_register('whatever I used')
}
```

This way, the outputted code will now automatically do all the necessary context switching, and can be easily called! So our CPU loop NOW looks like:

```typescript
while (cycles_left > 0) {
    block = get_next_block(core);
    func_ptr = block.instructions;
    func_ptr(core);
    cycles_left -= block.cycles;
}
```

### Congratulations you now have basic JIT
I hope you see that JIT isn't actually so very hard! And I hope you understand that there are a LOT of optimizations that can be made from here.

Some basic ones:
 
* Generate more of the code. Like generate opcodes for R3000_fs_reg_write instead of calling out to it!
* Write a scheduler that can run more than one block at once. Then optimize inter-block jumps so they don't need as many expensive thunks!
* Take that last part one step further, and write it in assembly code, meant to be jumped to between blocks. Perhaps zero registers will need to be pushed and popped after you are done, except for the beginning and end of a long chain of block executions!
* Instead of just keeping a pointer to CPU state (the core variable), put specific frequently-used registers directly into native registers, and extract them afterwards. This is very expensive but if you're running a bunch of blocks at a time, it is much less expensive than querying a data structure constantly!
* Use an intermediate representation to do inter-block optimizations of your generated code!
* Keep going to your heart's content! Just try to make it easy to debug

Debugging JIT is a whooooooole other challenge I can't even get started on here. One approach to start yourself off is that you allow two parallel CPU cores: one interpreted, one JIT. And compare their state between blocks. There are other ways to find different issues, but that's way outside scope here.

Have fun with JIT!
