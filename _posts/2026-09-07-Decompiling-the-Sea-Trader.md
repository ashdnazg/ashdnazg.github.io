---
layout: post
title:  "Decompiling the Sea Trader"
date:   2026-09-07
visible: 0
---

The Sea Trader is a great educational game and you probably belong to one of two groups:

1. You consider it a classic of the early 90s

    or

2. You've never heard about it until today

This polarisation is due to the fact that the game is fully in Hebrew, which rather limited its audience.

This article is about my 5 year (on and off) journey reverse engineering the game, culminating in a full matching decompilation[^1] of the game.

But to really understand this journey we need to go further back in time.

## 30 Years Ago

It's the middle of the nineties, I'm in primary school and we have a computer at home. As the youngest kid, I probably spend more time watching my older brothers play than I get to play myself. Not to mention, screen time is limited, but there's a loophole - educational games don't count into the same limit as [Golden Axe](https://en.wikipedia.org/wiki/Golden_Axe_(video_game)), [Digger](https://en.wikipedia.org/wiki/Digger_(video_game)) or [Civilization](https://en.wikipedia.org/wiki/Civilization_(video_game)).

Normally, this wouldn't be as much of a loophole, since educational games tended to be bad, but there was one notable exception - Socher Hayam (סוחר הים) - or in English - The Sea Trader.

In this game you have 7 days to make the most money by sailing your boat between three ports - Turkey, Israel and Egypt, buying and selling copper, olives and wheat. Every day you have new prices and in many voyages you can encounter random events such as pirates, storms, finding cargo on a deserted ship, pirates, navigational errors and pirates.

And through playing this game and watching my brothers play this game I learned valuable skills, like calculating the maximal profit in a trip, ending the day in a spot with a low price to stock up in expectation of a price increase tomorrow, and most importantly, paying for 3 guard ships every singe trip (the  mediterranean had quite the pirate problem).

Over the years games came and went, but every now and then I'd fire up the good ol' Sea Trader (this time in [DOSBox](https://www.dosbox.com/index.php)) to try and beat my high score.

## 6 Years Ago

It is the autumn of 2020, COVID is wreaking havoc all over, and I recently moved in with my girlfriend.

One day I decided to introduce her to the Sea Trader, which prompted her to start writing an AI[^2] for the game. Now, that necessitated having a good model of the game, specifically the distribution of the random daily prices.

I always assumed the prices had a uniform distribution, but assumptions are an unacceptable way to do statistics, so we went through a few games and painstakingly collected real data about the prices (mostly uniform, but there are special events).

That's when I told my girlfriend: "Why do we have to deduce the game's logic from statistics when we could just check the game's logic directly by reverse engineering it?".

To put things in context, just a year earlier [Ghidra](https://github.com/NationalSecurityAgency/ghidra) was released to the public, and I reverse engineered a game to [show how it violated the GPL](https://ashdnazg.github.io/articles/19/Finding-Evidence-for-GPL-Violation). Reverse engineering was more accessible than it had ever been before, so how hard could it be?

## Hitting Difficulties

Ghidra is a great tool if you're doing modern C/C++ decompilation, but the Sea Trader is neither modern nor was it written in C.

Your best friends when reverse engineering are strings, so I naturally started by looking for strings in the executable. It was very quickly visible that these are not [null-terminated](https://en.wikipedia.org/wiki/Null-terminated_string), but [length-prefixed](https://en.wikipedia.org/wiki/String_(computer_science)#Length-prefixed), a.k.a Pascal strings[^3].

Unlike most executables where strings are relegated to the dark corners of the executable, away from the code, here the strings seemed to be interleaved with the code, which confused Ghidra to no end.

Soon I noticed that before every such string there is always the same function call, which received the tentative name `LoadFollowingString`. After hacking together a python script for Ghidra that finds these calls, marks the following strings and fixes the flow of the program to skip them, Ghidra managed to make more sense of the code, but the decompilation was still barely readable.

Due to the fact that my 16-bit x86 assembly skills were non-existent, my progress was slow. Ghidra itself had difficulties dealing with the peculiar calling conventions in the executable and the ubiquitous use of [segments](https://en.wikipedia.org/wiki/X86_memory_segmentation).

Nevertheless, and intermittently over a period of years, I managed to make some progress. I identified the functions responsible for the daily events, for the random trip events and even found the one controlling the daily prices (what started this entire quest, if you remember).

I couldn't really understand what's going on in most of these functions, but I did have one interesting breakthrough, I've found a function with the following body:

```nasm
MOV        BX,word ptr [0x1fe]
MOV        CX,word ptr [0x1fc]
PUSH       BX
PUSH       CX
MOV        AL,BH
MOV        BH,BL
MOV        BL,CH
MOV        CH,CL
XOR        CL,CL
RCR        AL,0x1
RCR        BX,0x1
RCR        CX,0x1
POP        AX
ADD        CX,AX
POP        AX
ADC        BX,AX
MOV        AX,0x62e9
ADD        CX,AX
MOV        AX,0x3619
ADC        BX,AX
MOV        word ptr [0x1fe],BX
MOV        word ptr [0x1fc],CX
MOV        AX,BX
RET
```

The main point of interest is the following 4 lines:
```nasm
MOV        AX,0x62e9
ADD        CX,AX
MOV        AX,0x3619
ADC        BX,AX
```

Here we are adding `0x62e9` to the register CX and then adding `0x3619` to register BX, taking into account the carry from the previous addition.

In other words, we're treating BX and CX, two 16 bit registers, as one 32 bit value, to which we add the number `0x361962e9`=`907633385`. Maybe this magic constant means something?

A quick web search leads to the paper ["Random Number Generators: Good Ones Are Hard to Find"](https://www.researchgate.net/publication/220420979_Random_Number_Generators_Good_Ones_Are_Hard_to_Find):

> Similarly, the random number generator in standard Turbo Pascal is a full period mixed generator defined by f(z) = (129z + 907633385) mod 2<sup>32</sup>

> Incidentally, the generator in ‘87’ Turbo is subtly different.

Another quick check in Wikipedia's entry on Turbo Pascal shows that version 4 of Turbo Pascal was released in 1987, so our game must have been compiled with an earlier version.

Looking at the executable again, I noticed one string I stupidly ignored until now: `Copyright (C) 1985 BORLAND Inc`. With Turbo Pascal 2 being released in 1984, it meant that the game must have been compiled with Turbo Pascal 3[^4].

## A Year and a Half Ago

A web search for "Turbo Pascal 3" brings up an amazing page containing a [thorough analysis of how Turbo Pascal 3.01A works](https://www.pcengines.ch/tp3.htm) (I'll refer to it from now on as the TP3 Analysis). As I was reading it, to maybe learn something that will help me make sense of the assembly, I came upon the following sentence (emphasis mine):

> TURBO Pascal isn't your usual compiler... The parser is interspersed with portions of the code generator, **and there is no optimizer**. Most compilers need multiple passes to do their work, but TURBO is a (faster) **single pass compiler**.

This gave me an idea - if the compiler is single-pass and it does no optimisations, the assembly code must be *relatively* close to the original source code. This should be the ideal situation to deduce the source code from the assembly, or in other words, decompile the program. The problem, however, is that as mentioned before, I barely understood the assembly, so how would I know if my decompilation is correct? The answer is to make my life harder and attempt matching decompilation.

Instead of accepting any source code which is functionally identical to the executable (which is hard to verify), matching decompilation requires source code which compiles to an executable which is binary-identical to the one we started from (which is much easier to verify).

But before writing a decompiler, one needs a compiler, or at least a usable one.

## Turbo Pascal 3

Back in the day, I'm sure Turbo Pascal 3's features were great - it had an inbuilt editor and it could compile programs and run them *in memory*. But to verify a decompilation is matching, one has to run the compiler very often, and I simply can't be arsed to start it, manually load the file, change the options to produce a COM file and only then compile, *every single time*.

Luckily, as part of the TP3 analysis, there is a program to generate a full documented disassembly of the compiler. With a bit of trial, a lot of error and some learning about DOS, I [patched](https://gist.github.com/ashdnazg/ce73d2618eeaf068f57ac1bc60889de7) the assembly to accept a file name from the command line, set all the required options, compile the file, and immediately exit. Exactly like modern civilised compilers do.

Now that the compiler was usable, I consulted the TP3 analysis and manual to figure out where to start.

## Starting From the End

The first thing I did was creating an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) of a Pascal program. Since the last time I touched Pascal was around 25 years ago, I went directly to TP3's manual for a refresher. Amongst the various keywords I noticed something very useful - the `inline` statement allows one to output binary data directly to the executable.

This gave birth to a plan - instead of building the source code little by little while matching more and more of the binary, we're going to start with a 100% match, albeit with our functions containing no lines of actual pascal - every line will be an `inline` statement. Then, we're going to write rules that convert `inline` statements to regular pascal statements while preserving the match. The advantage is that every such conversion rule is applied on the entire executable, so if it works the first 5 times but causes a mismatch at the 6th, I can detect it and fix it immediately.

Once again, the TP3 Analysis was instrumental in documenting how functions look in binary, and before long I managed to detect the different functions[^5] in a short test program and generate a Pascal file containing said functions with their bodies replaced with `inline` statements. Compiling that file resulted in a 100% match (yay!). I went to try it on the Sea Trader's executable, and compilation failed (nay!). Apparently the resulting source code file was too big for TP3, but after splitting it to a few parts compilation succeeded with a 100% match (yay!)[^6].

From now on the process was more or less:
1. Find an assembly instruction that isn't decompiled yet.
2. Figure out what it's doing based on reading, neighboring instructions, the TP3 analysis, test programs, etc.
3. Add handling for that instruction.
4. Check whether this handling preserves the matching decompilation (usually it didn't)
5. Find the first place where there's a binary mismatch and return to step 2.
6. Repeat until the 100% match is restored.

If this sounds tedious, that's because it's absolutely is[^7]. So I hacked my motivation by making the decompiler print the percentage of bytes that are succesfully decompiled to Pascal statements.

## A Year Ago

Despite the aforementioned motivation, progress was not very fast. The breakthrough came in the shape of a deadline - my second kid was due soon and I figured that if it was hard fitting decompilation into my schedule with one kid, doing it with two (one of which is a newborn) is going to be even harder.

A few nights of little sleep and lots of coding pushed me into the 90%. There I started compromising on my principles. Until this point I tried to make my decompilation rules as general as possible and not deliberately overfit to my specific executable, but drastic times require drastic measures, so I allowed some logic which only worked in the specific circumstances of the Sea Trader's code. Towards the end I was willing to implement any hacky solution as long as it pushed the decompiled % up.

A few more nights and it was finally done, mere days before my kid was born. 3554 line of Pascal stood in front of me, with 0 `inline` statements!

At long last, I could read the game logic, just as I stupidly proclaimed to my then girfriend, now wife, 5 years earlier. Well, maybe read, but not understand...

## Func31_var_173

While the decompiler can create source code which compiles to the executable, it has no idea what this code actually does, so all functions and variables had nondescriptive names such as `global_var_2112` or `Func17_arg_2` (whose type is `arr_record_1479674575002027080_10`, if you wondered).

So a few more days/nights went to giving the appropriate names to everything, mostly helped by the strings interspersed all over the source code.

Finally, I could read *and* understand the game logic, and you can too: [https://github.com/ashdnazg/socher](https://github.com/ashdnazg/socher)

## The Friends We Made Along the Way

The reason I started this decompilation was because I could barely make sense of the 16-bit assembly, but by the end I was reading it fluently. Some of the hairier cases forced me to delve deep into TP3's (annotated) assembly to figure out what the hell was going on.

So both the original reverse engineering attempt and the decompiler required me to build the same knowledge, but the gradual nature of writing the decompiler was a much better framework for learning and that's why it succeeded.

While the decompiler won't likely work out of the box on other Pascal programs (I tried on a few and it mostly crashed), I believe it would be a great starting point, so if you have a program compiled by Turbo Pascal 3.x, give me a shout and I'll be more than happy to help you making it work.

Nowadays when decompilation projects of dubious accuracy materialize out of thin air every other day, validation is more important than ever, and I believe the answer is requiring matching decompilation.

---

[^1]: Source code that compiles to the exact same binary as the original.

[^2]: An offline AI, i.e., you input the information from the game and the program tells you what to do.

[^3]: There was also the issue of encoding, since all of the text is in Hebrew, which was found (by trial and gibberish) to be [Code page 862](https://en.wikipedia.org/wiki/Code_page_862) and written in reverse, since [Bidirectional text](https://en.wikipedia.org/wiki/Bidirectional_text) wasn't really a thing back then.

[^4]: Although the first Back to the Future film was released in 1985, so you never know...

[^5]: Here I learned another reason why Ghidra had trouble making sense of the file. Pascal has nested functions, which TP3 outputs *inside* their containing function.

[^6]: Well, a few bytes were not identical, none of them depending on the source code - see [Why are Some Bytes Ignored in the Validation?](https://github.com/ashdnazg/socher#why-are-some-bytes-ignored-in-the-validation)

[^7]: Remember that Ghidra had trouble with the calling convention? That's because there were none! It felt like every standard library function expected its arguments in different registers and/or on the stack. Almost every such function had to be handled separately by referring to the annotated disassembly of TP3.
