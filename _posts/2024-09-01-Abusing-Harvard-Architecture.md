---
layout: anchored_post
title:  "Abusing the Harvard Architecture of Nand2Tetris"
date:   2024-09-01
visible: 0
---

Every now and then, when one writes assembly code by hand, one notices opportunities for optimisation.

Some are concise and elegant, it's usually a good idea to use those.

Some are complicated or convoluted, you might use them if the benefit they bring is significant.

Then there are those who are outright ludicrous. You'd never use these in any production code, but if you think of one, you're definitely going to jokingly suggest it to your coworkers just to see their horrified faces. Or maybe write about them in a blog post.

<!--more-->

## Nand2Tetris?

_I attempted to write this post assuming no prior knowledge of Nand2Tetris. In case you are familiar with the course and its assembly language, feel free to skip to [the Challenges](#the-challenges).

[Nand2Tetris](https://www.nand2tetris.org/) is a wonderful course where one learns the basics of hardware of software from the ground up by designing a CPU and writing a compiler stack for it, culminating in the development of an application e.g., Tetris.

Normally, one goes through this course once (perhaps as part of a degree), learns a bunch and never touches any of the material again, as the CPU, the assembly language and the compiler stack are all invented by the course and don't occur in the wild.

Nevertheless, more than 10 years since I finished the course, I still find myself playing with the Nand2Tetris assembly, and apparently [others do as well](https://medium.com/@MadOverlord/optimizing-nand2tetris-assembly-code-f378700d0096).

<!-- "there are dozens of us" meme?  -->

## Why Nand2Tetris Assembly?

You're probably wondering why working with Nand2Tetris assembly can be more interesting than working with something that does exist in the real world, such as x86 or ARM.

It is precisely the disconnect from reality that makes it appealing.

First, as an educational tool the CPU is very simple in design. Depending on how you count, it has ~19 documented opcodes ([and 14 undocumented ones](https://medium.com/@MadOverlord/14-nand2tetris-opcodes-they-dont-want-you-to-know-about-f3246831d1d1)). Compared to real world alternatives, this is a very low knowledge barrier - you don't have to remember countless options nor constantly consult an online reference.

Second, unlike real world architectures, there is only one implementation of the CPU. Sure, different people write slightly different designs, but they all execute exactly one instruction per clock cycle. That makes instruction count a great metric for how optimised an algorithm is. Compare this with x86 and its countless processors, each having vastly different performance characteristics for different instructions.

## Nand2Tetris CPU Architecture

I'm not going to go into the fine details of the architecture, but luckily, as I mentioned before, there's also not that many details to begin with.

It is most likely that you're reading this on a device using the [von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture) where the data and the program all reside in the RAM, [hopefully in different addresses](https://en.wikipedia.org/wiki/Vulnerability_(computer_security)).

Our CPU, on the other hand, uses the [Harvard architecture](https://en.wikipedia.org/wiki/Harvard_architecture). In this architecture there are separate memories for data and for the program, in our case they are named [RAM](https://en.wikipedia.org/wiki/Random-access_memory) and [ROM](https://en.wikipedia.org/wiki/Read-only_memory) respectively.

The CPU has two 16 bit registers:

1. `D` - Multi-purpose register. Doesn't have any special uses.
1. `A` - Load/Pointer register:
    1. Can load any positive 16-bit integer value.
    1. Can be used as a pointer for reading/writing values from/to the RAM.
    1. Can be used as a pointer to an instruction in the ROM to which the program can jump, thereby implementing flow control.

In addition there's a fake register, `M`, representing the contents of the memory address pointed to by A (you can think of it as `*A`).

The instructions come in two types as well:

1. Load Instruction - Sets `A` to a positive 16 bit integer. Its assembly representation is `@<integer>`, e.g., `@7447` or `@32223`.
1. Compute Instruction - performs a computation on the 0-2 registers, optionally stores the result into any combination of registers and optionally jumps to the instruction in the ROM pointed to by `A`. Its assembly representation is \
`[<destination_registers>=]<computation>[;jump_condition]`,\
 e.g., `AD=A+D` (set `A` and `D` to their sum), `D;JEQ` (jump if `D` equals 0), `D=D+1;JMP` (increment `D` by 1 and then jump unconditionally).

And that's it!

## The Challenges

As you might have noticed, the A register has too many hats. Whenever we want to access the memory, `A` needs to hold the relevant RAM address. Whenever we want to implement any type of flow control, `A` needs to hold the relevant ROM address.

In practice, that means `A` can't store any values long-term, since `A` needs to change all the time. Let's look at the following example implementing an absolute value operation:

```
// The program calculates the absolute value of RAM[0]
// and writes it into RAM[1]

@0                  // Load 0 into A
D=M                 // Load the value in the RAM in the address A into D
@IS_POSITIVE        // Load the ROM address of (IS_POSITIVE) into A
D;JGT               // if D >= 0 then jump to the address in A
D=-D                // Negate D
(IS_POSITIVE)       // Label referring to the next instruction
@1                  // Load 1 into A
M=D                 // Store D into the RAM in the address A
```

Even in such a short code excerpt, we had to write 3 different values to `A`.

The reality is actually grimmer. Pay attention to how the jump condition checked whether `D` is greater than 0. Remember that when jumping, `A` is reserved for the jump target, therefore it cannot be used for the condition. `M` can't be used either, as it depends on `A`, and since `A` is reserved, `M` is the contents of whatever arbitrary RAM address `A` points to, which is most likely not the one you'd actually like to check.

This means that `D` _is_ actually special as well, as it needs to be used for jump conditions.

The consequence is that when implementing any non-trivial algorithm, you find yourself spending most of the instructions putting values into and out of the RAM as the registers are always needed for something else.

Optimisation is then focused on removing this unnecessary overhead.

## Optimising Multiplication

