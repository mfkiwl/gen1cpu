# Design & Planning

## Issues with other platforms

ARM has always been a pain in the arse (the peripheral ABIs are horrendous, the tools are terrible, libraries made available by chip manufacturers are generally *the absolute worst*, etc.). (Intel) x86 is an absolute mess of half-working backwards compatibility features, both appear to have ongoing security issues, and alternatives like MIPS, POWER and SPARC seem to basically be dead or unclearly-specified and unclearly-licensed. All are over-optimised for running 90's code and the 64-bit versions are probably all a bit hacky as a result (too many legacy/compatibility features, too many ABI versions, too many tool options, etc.).

I originally was a bit confused by RISC-V, since it seemed to be slowly taking over the world but (looking at the specs) it seemed a bit half-baked and unfinished, so I started looking for alternatives (mostly for use in robotics). I'm frankly kind of surprised anyone managed to port operating systems to it because I just didn't see a clear structure for that kind of stuff (it's probably starting to mature now, but I guess the buzz just started way before the system-mode ISA was finalised).

In any case, RISC-V seems to be the kind of system that probably makes sense to hardware designers more than programmers (one example is the use of a hardcoded zero register - it "makes sense" in the context of RISC ideals, but it just doesn't make sense from a design perspective - registers and constants are totally different ideas and should never be mixed up!). So, I'm not trying to discount it's relevance for teaching CPU concepts (and - possibly - it's relevance in efficient real-world chips), but it didn't seem to be what I was looking for as a stable and future-proof platform for robotics projects (in particular, simplicity and extensibility matter more than instruction efficiency).

Conventional embedded chips like those used in Arduinos are fine (or at least the Arduino SDK solves many of the chip-specific problems), but the more robust chips can't really run large/modern programs and for lower-level code they typically face similar problems as other alternatives.

### A New Instruction Set

So, I decided a simpler solution was required. Initially I hired a third-party developer who was familiar with MIPS to design a simple CPU core, and the developer completed this with minimal issues. I knew MIPS wouldn't be sufficient, especially since most versions of the instruction set aren't open-source, but I managed to use what I'd learnt from working with this developer's implementation to develop a CPU for my own instruction set. My implementation isn't nearly as fast as the original but is also designed to nicely handle edge cases such as hardware interrupts, bus errors and invalid instructions.

The new instruction set is still somewhat similar to MIPS/RISC-V/ARM but not as "clever" in it's encoding, which makes developing tools and bootstrapping implementations a little easier and should also avoid infringing on any copyright (or worse, patents) for such cleverness. Luckily, computers don't need to be very clever (unless you need to suck up to investors), and often encoding-level optimisations aren't critical to modern use cases (or such optimisations make more sense as extensions). Using simpler encodings also allows them to be made slightly more memorable and easier to encode/decode by humans, which helps for the purposes of developing and debugging low-level code. 

Obviously existing CPUs and MCUs have a lot of extra features (FPUs, MMUs, often built-in GPUs, etc.) and are already very fast, so there's not really much point trying to compete in terms of features-per-chip or gigahertz-per-chip or instructions-per-cycle etc. at least in the first generation of a new architecture, but I've tried to improve upon them in terms of ease-of-use: The core is defined in a single Verilog file (which should basically "drop right in" to a project with any FPGA development kit), the instruction set is easier to read/write/debug in hex format, and the bus wires and bus instructions are simplified to a reasonable form which can be extended or restricted as necessary for optimised implementations.

The instruction set is documented chiefly in the main Verilog file, but also under [InstructionSet.md](InstructionSet.md).

## Why So Many Registers?

Actually, a few reasons.

1. Registers are more secure than external memory (if you're not worried about your motherboard leaking your cryptography secrets, then you're probably not a security expert)
2. Using an encoding of either 16 or 256 registers makes the instructions easier to read (and if there's free space anyway - why not just use it to allow 256 registers? on the other hand in some instructions we only allow the lower 16 to be used anyway, so it's not like we're really wasting any encoding space)
3. If anyone is like "nah this other CPU architecture is faster" I can be like "well, in theory... this architecture could do the same thing with less memory reads/writes, so, in theory..." (in other words, more registers gives a natural speed advantage, but there are tradeoffs in space and power usage meaning you might not want too many in hardware)
4. It allows for future implementations to have more flexibility over registers. For example, the operating system might want to define which registers are enabled/disabled in software so that it can cache things more efficiently, or an application might want to keep one register holding a "context" value at all times for debugging or to simplify some API internals.

As for standardisation, however, a program can generally assume that there are at least four working registers, and that in a high-level environment higher registers may be accessible anyway (but may not be as fast as lower registers, since they might be emulated or controlled for security/multitasking purposes). For C tools, I'm currently assuming there are 16 registers but limiting usage of those beyond `$r7`.

## Why A Separate I/O Bus?

Nowadays, most systems access I/O devices through the memory bus instead of dedicated I/O ports. Why? Because "it's simpler" and "it works better with C" and "we can just use an MMU to multiplex everything".

However, that may not be an appropriate option for high-security environments. If all your device access happens via a complex MMU with cacheing and other features, all your memory access happens via the MMU, and all your cryptography happens via the MMU, then any device with access to the MMU can potentially compromise all your devices, all your memory and all your cryptography. So although the current hardware design still uses the same wires to save space, the design of the instruction set allows more flexibility in situations where you want to isolate components as much as possible.

A separate I/O bus with a complete memory-like interface not only allows us to separate I/O access from "application memory", but it even allows us to forego the MMU entirely. System code can just store it's data via the I/O bus (which can also just be connected to memory) whereas user code can use the memory bus (which could be connected to totally separate memory) and instruction code can be stored in a separate ROM. You could even use one ROM for system code and normal memory for application code, since the system-mode flag is also part of the bus.

I repeat: On this platform, you do not need a complex and error-prone MMU in order to efficiently isolate system and application code. An MMU might still be handy for heavy multitasking workflows, but it isn't strictly required to implement process isolation in multitasking environments. The built-in Real-Time MMU is designed to fit common use-cases (including stack protection, Address Space Randomisation and other security-related concerns), however an external MMU can still be added by implementors if necessary.

## Future Plans

Most updates at the moment are happening in the tools, particularly bootstrapping C (which will probably take a bit more fine tuning). Aside from compiler-like tools, adding a testing/verification suite and a standard microcontroller implementation would be next major issue (right now testing on hardware is mostly done by making a kind-of dummy microcontroller to check instructions by flicking lights - this works well enough to check that an instruction "seems to work" but is a kind of primitive and limited approach!). Ideally, a single test suite (perhaps with different options) could be used for testing a variety of hardware and simulated implementations.

As for the CPU itself, additional peripherals like a floating-point unit (FPU) would probably be desirable and many internal optimisations are also possible. An obvious optimisation would be to reduce the number of internal stages as much as possible (closer to a conventional RISC design), but this may make it more difficult to add new instructions.

The basic design should also be applicable to 32-bit implementations but this isn't included as an option in the Verilog code yet. An earlier design included an option to change the word and address size but it became more difficult to test since switching modes impacted all the testing scripts too (once the design stabilises it should be easy to add a 32-bit option).

The current compromise is to use a 32-bit data and instruction bus (with 32-bit instructions) _but a 64-bit address bus, registers, ALU, etc._. This should be reasonable for most uses since the upper 32 bits of the address bus can just be ignored in hardware (i.e. if the hardware has a 32-bit external bus), whereas 32-bit fetches would be required for instructions anyway (so even on a 64-bit external bus would require some way to fetch 32-bit values). However, this compromise probably isn't perfect for any use case - for example if it's surrounded by a cache layer that layer will probably want to allow 64-bit fetches regardless of the external bus, and 64-bit fetches would probably need to be provided in software if the hardware doesn't support them; conversely - a smaller implementation would probably benefit from having 32-bit internal registers and ALU anyway regardless of external bus. A more robust solution might be to generate different variations with a program rather than to try to work all of them into a single Verilog implementation.

## TODO

* Obviously a better name ("gen1" is just CPU-speak for "first-generation design")
* More advanced tools (work-in-progress...)
* FPU (could be implemented by extending the control functions, but would likely take up a lot of FPGA space)
* ~MMU (could possibly be implemented over the current bus though)~
* Optimisation of existing instructions (most/all instructions take more cycles than should be strictly necessary)
* Addition of optimised instructions (e.g. you can already read an 8-bit or 64-bit value from memory, but it will currently take more instructions than just reading a 32-bit value)
* Test cases (I have done some ad-hoc testing, initially with Icarus Verilog and also on FPGA, but only the FPGA module is included as I mostly trashed the other one while learning Verilog)
* etc.
