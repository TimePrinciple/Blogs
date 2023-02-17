# The Secret Life of Programs

## Introduction

Learning to code is only a starting point: it's not all that difficult to write computer programs that *appear* to work, or work much of the time.

This sounds great in theory, but it's not a great idea in practice.

Understanding underlying technologies helps you develop a sense of what can go wrong. Knowing just high-level tools makes it easy to ask the wrong questions. It's worth learning to use a hammer before graduating to a nail gun.

Be really careful how to phrase it (to computers).

It involves finding elegant solutions as opposed to using brute force. Yes, you can get out of a house by bashing a hole in the wall, but it's probably a lot easier to go out the door.

Application programs don't talk to the computer hardware directly; that's where system programming comes into play. System programmers make the building blocks used by application programmers. System programmers need to know about hardware because their code interacts with it.

Each level (layer) builds on the one beneath it. This means that poor design choice or errors at lower levels affect everything above.

System programming is at the bottom of the software hierarchy.

System programming is sandwiched between application programming and computer hardware, which means you need to learn something about both of those.

## Chapter 1

Language only works if all communicating parties have the same *context*, so they can assign the same meaning to the same symbols.

Computers have a *condition code register*, which is a place that holds odd pieces of information. One of those is an *overflow bit*, which holds the carry value from the most significant bit. We can look at this value to determine whether or not overflow occurred.

Two's Complement notation is meant to get rid of `-0` notation by trying to get 0 by adding `1` and `-1` (000 0001 + ? = 1 0000 0000) to `0` yields the compliment + 1 notation of negative numbers (thus ? is 1111 1111), then the `-0` notation `1000 0000` is given to `-(MAX + 1)`.

**Creating a new context for interpretation.**

Hexadecimal representation (meaning base-16) has pretty much take over because the insides of computers are built in multiples of 8 bits (8-bit chunks as a fundamental unit, which called a byte) these days, which is evenly divisible by 4 but not by 3 (the Octal).

The term character is often used interchangeably with byte because characters' codes have typically been designed to fit in bytes.

Unicode Transformation Format 8-bit (UTF-8)

URL encoding, also known as percent-encoding, replaces a character with a % followed by its hexadecimal representation.

Specifying colors using hex triplets. A hex triplet is a # followed by six hexadecimal values formatted as #rrggbb where rr is the red value, gg is the green value, and bb is the blue value.

## Chapter 2

The scale is what mathematicians call continuous, meaning that it can represent read numbers. The fingers, on the other hand, are what mathematicians call discrete and can only represent integers.

The word analog to mean continuous and digital to mean discrete.

*Continuous* meaning that it can represent real numbers. *Discrete* can only represent 'integers'.

*Propagation delay* is the amount of time it takes for a change in input to be reflected in the output.

## Chapter 3

To minimize the chances of getting incorrect results, use the transition between logic levels to grab the data instead of grabbing it when the logic level has a particular value. (same concept as difference signaling)

**Asynchronous** means everything happens when it gets around to it. The problem with asynchronous systems is that it's hard to know when to look at the result. (due to *cascading effect*, it takes time for the output to change as the scale of cascaded components) 

Register, which are a bunch of D flip-flops in a single package that share a common clock, thus it's **Synchronous**.

Stated by the great performance pioneer Jimmy Durante, best performance is a-ras-a-ma-cas. This is a very important consideration in programming: keeping things that are used together in the same row greatly improves performance.

Because flash memory wears out, solid-state drives include a processor that keeps track of the usages in different blocks and tries to even it out so that all blocks wear out at the same rate.

Use *voting system* for decision making or error detection.

ECC: Error Checking and Correcting.

CRC: Cyclic Redundancy Check.

## Chapter 4
