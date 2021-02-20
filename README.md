# gba-hq-mixer (needs testing)

This repository contains my so called "HQ mixer" for GBA games which use the m4a/mp2k sound driver.

## Back Story

Most games that use Nintendo's m4a/mp2k sound driver have a common problem of rather noisy sound.
This is because on the GBA PCM sound has to be entirely generated by the CPU since there are only two PCM hardware channels (usually used for left/right stereo output).
Nintendo's sound driver processed 8 bit PCM sample data, resamples it to the engine's output rate, applies volume scaling, and "reverb".
All of this processing is done with the same buffers which are used for the hardware channels via DMA.
Since these buffers are 8 bit resolution only, directly mixing on this buffer has inherent noise problems since each additional processed sound generated additional quantization error.
This is also why games with lower simulateous voices usually are less noisy than games which use more.

When making hacks/mods for GBA games, it is often desirable to use more voices for higher fidelity music, however, the music will inherently become really noisy.
To solve this problem, I started developing a new sound processing routine back in 2014 which aims to solve all the noise problems.
It's gone through quite a few iterations since then, so if you've been using it back then, the newer codebase might be of interest.

Parts of the code are inspired by reverse engineered code from Golden Sun's sound driver, which is also a modified version of Nintendo's sound driver.
Golden Sun is well known for featuring one of the highest quality music released in a commercial GBA game. It caught my attention since the code did have much lower noise than most other games.

The solution to avoid the noise is to use a higher resolution buffer during all the processing. My code uses a 16 bit PCM buffer internally.
Higher resolutions can also be used but practically wouldn't be very beneficial (other than wasting more RAM).
By using a higher resolution buffer, quantization error is only introduced a single time, at what I call "downsampling" stage.
This is an extra step necessary by this code, which converts all the 16 bit audio into 8 bit after all major processing is done.
It might initially not be as intuitive, but introducing quantization only a single time (in opposite to doing it for every voice played) practially does reduce the audible noise quite significantly.

After writing the code initially, another feature appeared to be quite beneficial: processing speed.

Even though Nintendo's code doesn't do anything terrible by regular programming standards, there are quite a few tricks which can be used to accelerate the sound processing. You can find more details on that below.
This may allow you to use more voices in your hacks/mods and/or to use higher samplerates for higher fidelity music.

## Features

- Lower noise than Nintendo's default code
- Faster processing speed (especially at higher samplerates)
- Support for additional features (Camelot's synth instruments, Pokémon's compressed samples, etc.)

## Configuration and Versions

The HQ mixer has different features and different versions.
Configurations are different `.equ` directives in the code which will affect the code's behavior.
Versions are different git branches of this code. The available versions can be seen in the list of branches.

Before you can use the HQ mixer, you have to choose a configuration and a version:

### Versions

- master: This is the one you most likely want (should be default branch on GitHub)

Depending on what folks might need, more versions with different feature sets might become available in the future. For now, only the single main version is available.

### Configuration

The assembly program has a little section right in the beginning with the following `.equ` directives which you should choose correctly for your case:

- `hq_buffer_ptr`: This should be defined to an IWRAM address (0x03000000 - 0x03007FFF). The amount of memory needed depends on the samplerate used. Total number of bytes required is `samples-per-frame * 4`.
- `POKE_CHN_INIT`: Pokémon games initialize the PCM channels a bit different, so you should set this to `1` if you use this on a Pokémon game. For other games, select `0`.
- `ENABLE_STEREO`: This option is not yet available and should be set to `1`. In the future this is intended to allow this code to be used with games that only output mono sound. You cannot simply use the standard stereo version on mono games without corrupting memory.
- `ENABLE_REVERB`: To get similar results in terms of reverb as default games, set this to `1`. If you don't need the reverb effect, you can increase processing speed by setting this to `0`.

## How to assemble

Since this assembly program is self-contained, you can simple compile it to it's binary form like this:

```
arm-none-eabi-as m4a_hq_mixer.s -o m4a_hq_mixer.o
arm-none-eabi-objcopy -O binary m4a_hq_mixer.o m4a_hq_mixer.bin
```

You can then take the binary file and copy it anywhere you need. The code is position independent, so no linking is required.
Of course you can also integrate the assembly program in your source code level project too. In that case you'll have to come up with your own way of integration.

## How to insert (binary hacking)

Inserting the code when doing binary hacking may be quite difficult, depending on what game you try it with.
Nintendo's m4a/mp2k driver has a function called `SoundMain` which is executed every frame to do all the sound processing.
After all sequences are processed, the PCM channels are processed.

In order to maximize performance for this timing critical step, this mixer code (which is what we'll replace) is copied to IWRAM and runtime.
This creates somewhat of a challange to replace the function since you can no longer simply replace a function in ROM and you're good.
Free IWRAM space is usually fairly limited in most games (32 KiB total) and it's not always obvious which space is free.

Since the original code also resides in IWRAM you might consider that the new code simply goes into place of the old one.
Unfortunately this usually does not work out since my code is generally a bit larger. So you'll have to find additional space for the code.
The final size required depends on the configuration, so you should assemble the file like above and check the actual number of bytes required.

One of the advantages of the new mixer is the higher bit depth mix buffer. This needs a place in IWRAM as well and can be configured with the `hq_buffer_ptr` definition.

After you've sorted all these things out and you have a location for all IWRAM addresses, you can proceed with the insertion process.
Place the assembled program somewhere in the ROM. This on its own is obviously not enough since the code is not called or copied to IWRAM.
Usually games have two three pointers which you'll have to edit:

- IWRAM call address
- IWRAM CpuSet destination address
- ROM CpuSet source address

Now it might seem intuitive that we have to specify the IWRAM and the ROM pointer for our new mixer code.
The problem is that the pointer literals of the engine's code are not global variables, so they appear in more than once place.
One such place is the initial copy from ROM to IWRAM. This is usually initiated from the function `m4aSoundInit` and copies the code from ROM to IWRAM using CpuSet from the BIOS.
Although it's practically not necessary for copying, the ROM pointer generally has the Thumb bit set while the IWRAM pointer does not.

After initialization the code also has to be executed. This is usually done from a function called `SoundMain` which will read a second IWRAM pointer.
This IWRAM pointer is identical to the one specified in `m4aSoundInit`, however, this IWRAM pointer does have the Thumb bit set since the entrypoint of the code starts with Thumb instructions.

If these three pointers are changed, your new code should be called and things should work just fine.
However, you can do some more advanced memory rearranging to free IWRAM which might require further pointer changes.
This will depend on the game you use and therefore I cannot give further instructions on what you should do here.

## Useful ASM Tricks

In this section I will explain briefly which tricks my HQ mixer uses to really squeeze every bit of performance out of the GBA's ARM CPU.

### Vectorization

Even thought GBA's ARM7TDMI CPU technically doesn't have any vectorization support, we can still apply some tricks which somewhat work that way.
One common operation of the mixer code is to fetch a signed 8 bit PCM sample, apply volume scaling for left and right speaker, and save the result back to buffer.
Let's look at the following bit of code briefly:

```
@ R10 = left volume (0-127, unsigned)
@ R11 = right volume (0-127, unsigned)
@ R12 = sign-extended 8 bit PCM sample (-128 to 127)
@ R5 = 16 bit destination buffer (left, right interleaved)

LDRSH  R0, [R5]     @ load mix buffer left sample
MUL    R1, R12, R10 @ multiply PCM sample with left volume
ADD    R0, R0, R1   @ ... add that to our sample from mixer buffer
STRH   R0, [R5], #2 @ save value back to mix buffer and step to next sample

LDRSH  R0, [R5]     @ same procedure for right sample
MUL    R1, R12, R11
ADD    R0, R0, R1
STRH   R0, [R5], #2

@ execution time: 16 cycles
```

Now, there are two things we can optimize:
First, we can obviously replace the `MUL` and `ADD` with a `MLA`.
This isn't faster but it saves one instruction in terms of code size.
Second, have a look at this:

```
@ R10 = 0x00RR00LL where LL corresponds to left volume, RR to right volume
@ R12 = sign-extended 8 bit PCM sample (-128 to 127)
@ R5 = 16 bit destination buffer (left, right interleaved)

LDR  R0, [R5]
MLA  R0, R10, R12, R0
STR  R0, [R5], #4

@ execution time: 8 cycles
```

Okay, what happened here? We suddenly only read/write one 4-byte value from the mix buffer.
So `R0` will contain both the left and the right 16 bit value of the mix buffer.
The left part will be in the lower 16 bit, the right part in the upper 16 bit (litte endian).
First, what happens at the multiplication part?
Due to the way the left/right volume are contained in `R10`, a multiplication with a scalar value will multiply both value at their position inside the register.

Let's go through the following example:

```
R10 = 0x00400020       @ right volume: 64 (=0x40), left volume: 32 (=0x20)
R12 = 0x6E             @ sample: 110 (=0x6E)
temp = R10 * R12       @ temp = 0x1B800DC0
left = lower16(temp)   @ left = 0x0DC0 (3520)
right = higher16(temp) @ right = 0x1B80 (7040)
```

You see how we just effectively did two multiplications with a single multiplication? Cool, huh!

Now, there is a catch: Let's redo the example with a negative sample value:

```
R10 = 0x00400020       @ right volume: 64 (=0x40), left volume: 32 (=0x20)
R12 = 0xFFFFFF92       @ sample: -110 (=0xFFFFFF92)
temp = R10 * R12       @ temp = 0xE47FF240
left = lower16(temp)   @ left = 0xF240 (-3520)
right = higher16(temp) @ right = 0xE47F (-7041)
```

So this time the multiplication for the right sample is not correct.
Perhaps you can figure out that if the sample is negative, you'll accidently subtract 1 from the right result.
Now, luckily for us this is not a dealbreaker. It might not appear immediate, but this calculation error is quite minor and will in practice not cause any audible difference (at least in GBA terms).
Though, in the end, execution speed for that section is effectively doubled.

### Loop Unrolling

If you're familiar with compilers, you may know this one: Unrolling usually refers to loop code being duplicated in order to reduce the amount of jumps executed.
Because ARM has instructions like `LDM` and `STM`, not only the number of jumps are reduced, but the loads and stores can be accelerated at the same time:

```
LDMIA R5, {R0-R3}
MLA   R0, R10, R4, R0
MLA   R1, R10, R5, R1
MLA   R2, R10, R6, R2
MLA   R3, R10, R7, R3
STMIA R5!, {R0-R3}

@ execution time: 19 cycles
```

In practice the code will do a little more than a single `MLA` but for this example we've executed the sample from above 4 times, but with only roughly 2.5 execution time increase.

### Self-modifying code

One problem that arises with loop unrolling is the increased code chunk size.
If you'd want to have an if/else around that loop, you'd have to duplicate that huge loop, even though only minor parts of the code may differ.
Now we have to remember that our code will reside in IWRAM at runtime. IWRAM's size is quite constrained, so we better should keep the code as small as possible.
So instead of duplicating large code blocks, we can simply modify parts of the loop body right before executing it.

This is why my mixer program makes extensive use of self modifying code in order to keep performance as high as possible but without exploding code size.
You may be able to spot these sections in the code where there is `NOP` placeholder instructions that'll get overwritten with real instructions at runtime.

### Know your instruction set

This section is not terribly exciting, but the ARM instruction set can do some really funny things.
When used correctly, you can pack a lot of functionality into a single instruction.

We've discussed `LDM` and `STM` previously, but just look at this monstrosity I've used in the code:

```
LDMEQFD SP!, {R0, R2-R5, PC}    @ only if zero flag is set, cleanup stack and return
```

Or this example that uses the barrel shifter in combination with a load instruction:

```
LDRNE  R0, [R5, -R8, LSL#2]
```

I'm particulary proud to have came up with the following one:

```
@ assume LR = 0xC0000000
ADDS   R4, R12, R12
EORVS  R12, LR, R4, ASR#31
```

What does the code above do? It clamps a signed 32-bit input value in R12 to effective 31-bit range.
This is particulary useful at the downsampling stage where we do this in order to clamp the output range instead of causing overflow pop artifacts.
This is how the code looked before:

```
CMP   R12, #0x3FFFFFFF
MOVGT R12, #0x3FFFFFFF
CMP   R12, #-0x40000000
MOVLT R12, #-0x40000000
```

### Pre-caching sample data

One last thing I want to mention is something about the GBA Bus and read/write speeds.
This is a small table on how fast reads and write actually are:

Instruction | Memory | Clock Cycles
--- | --- | ---
LDR(S)B | ROM  | 6
LDR(S)B | IWRAM | 3
LDR(S)B | EWRAM | 5
LDR(S)H | ROM  | 6
LDR(S)H | IWRAM | 3
LDR(S)H | EWRAM | 5
LDR | ROM |  8
LDR | IWRAM | 3
LDR | EWRAM | 8
STRB | ROM  | N/A
STRB | IWRAM  | 2
STRB | EWRAM  | 4
STRH | ROM  | N/A
STRH | IWRAM | 2
STRH | EWRAM | 4
STR | ROM |  8
STR | IWRAM | 2
STR | EWRAM | 7

As you can see it is quite relevant which instruction you choose to do what.
Let's say we want to read four 8-bit values from ROM:
Do we do four times LDRSB oder we do a single LDR and do the sign extension afterwards?
Four LDRSBs from ROM would be 24 clock cycles while doing a single LDR would only take only 8, however, then you manually have to do sign extension.

In practice doing the latter is the more efficient way but requires more complicated code.
In some parts of my code that would make the code really complicated.
Because of that, we can leverage the extra speed of LDRSB being faster on IWRAM than on ROM.
Though, the data has to get to IWRAM first (we'll use the main stack for storage), so doesn't this extra copy make the code slower?

Let's do some calculation for a chunk of 256 samples loaded from ROM to IWRAM via DMA3 (which can do even faster sequential transfers):
The number of cycles used for a DMA transfer is the following:

1. non-sequential read from source (ROM): 6
2. sequential read from source (ROM): 254
3. all writes to destination (IWRAM): 64
4. sample fetching (IWRAM): 768

Which adds up to 1092 cycles. Now, reading the samples from ROM directly is much simpler and only has one step:

1. non-sequential read from source (ROM): 1536

One of the main problems is that by reading any non-sequential byte from ROM, we actually always fetch one word which is more than we need, however, it cannot be avoided due to the way the ROM bus works.

So as you can see, using the buffering technique, we only need ⅔ of the processing time.
