# SWEET32-65816
A SWEET16 inspired 32-bit VM for the 65816
(SW32 for short)

This CA65 Library implements a 32-bit RISC Processor in software for the 65816, which can then be used to more easily handle 32-bit functions (or if you're just more familiar with RISC-like Instruction sets)
It has the following features:
* 15x 32-bit General Purpose Registers R1-R15 (R0 is a constant 0)
* 32-bit Program Counter (though functionally it's only 24-bits wide)
* 8-bit Status Register called SR, with 3 ALU FLags (Zero, Carry, Negative), and 4 user programmable Flags F4-F7
* 24-bit Addressing for both Program flow and Load/Store Instructions

Almost all Instructions are 2 Bytes long, except for the "Load Immediate" Instructions which are either 4 or 6 Bytes in length.

# Using the Libary
It should be as simple as including the `SWEET32.inc` file in your main source file, which defines all the macros for the VM's Instruction set and include the `SWEET32.lib` file when Linking with ld65 (just make sure it's located after (ie to the right) your source files in the ld65 command)

in case you need/want to modify the stock VM, the source files are simple enough to assemble and combine into a `.lib` file like this:
```
CA65 -v --cpu 65816 -o sw32_init.o sw32_init.a816
CA65 -v --cpu 65816 -o sw32_execute.o sw32_execute.a816
CA65 -v --cpu 65816 -o sw32_print.o sw32_print.a816

AR65 v v r SWEET32.lib sw32_init.o sw32_execute.o sw32_print.o
```

# Instruction Set
The Format is pretty simple:
Ra = Source Register A
Rb = Source Register B
Re = Destination Register
Rx = Source and Destination Register
|Name|Mnemonic|Description|
|---|---|---|
|Branch on Clear|BNv k|Branches if the specified bit "v" in the SR is 0<p>"k" is an 8-bit signed offset multiplied by 2, giving it a -128 to +127 Word range|
|Branch on Set|Bv k|Branches if the specified bit "v" in the SR is 1 (Offset is the same as above)|
|Jump and Link|JAL Re, Ra|Jumps to the Address in the Source Register and stores the address of the following instruction in the Destination Register<p>using R0 as the Destination Register makes JAL function like a regular Jump, using R0 as the Source Register won't modify the PC, bascially just loading the Address of the next Instruction into a Register|
|Add|ADR Re, Ra, Rb|Re = Ra + Rb, updated Flags: Z, C, N|
|Subtract|SBR Re, Ra, Rb|Re = Ra - Rb, updated Flags: Z, C, N|
|Logic AND|ANR Re, Ra, Rb|Re = Ra AND Rb, updated Flags: Z, N|
|Logic OR|ORR Re, Ra, Rb|Re = Ra OR Rb, updated Flags: Z, N|
|Logic XOR|XOR Re, Ra, Rb|Re = Ra XOR Rb, updated Flags: Z, N|
|Logic Shift Left|SFL Re, Ra|Re = C <- Ra <- 0, updated Flags: Z, C, N|
|Logic Shift Right|SFR Re, Ra|Re = 0 -> Ra -> C, updated Flags: Z, C, N|
|Rotate Left|RLL Re, Ra|Re = C <- Ra <- C, updated Flags: Z, C, N|
|Rotate Right|RRR Re, Ra|Re = C -> Ra -> C, updated Flags: Z, C, N|
|Add Immediate|ADI Rx, k|Rx = Rx + k (8-bit, sign-extended to 32-bit), updated Flags: Z, C, N|
|Load Byte (Signed)|LB Re, Ra|Uses the contents of the Source Register as an Address to load a Byte, sign-extend it, and store it into the Destination Register|
|Load Byte (Unsigned)|LBU Re, Ra|Same as above, except it zero-extends the Byte|
|Load Word (Signed)|LW Re, Ra|Same as the first, except it loads a 16-bit word and sign-extends it|
|Load Word (Unsigned)|LWU Re, Ra|Same as the first, except it loads a 16-bit word and zero-extends it|
|Load Long|LL Re, Ra|Same as the first, except it loads a full 32-bit word, so no "extending" is required|
|Store Byte|SB Ra, Rb|Uses the contents of the Source Register A as an Address, and stores the Low Byte of the Source Register B to that Address|
|Store Word|SW Ra, Rb|Same as above, except it stores the Low Word of the Source Register B|
|Store Long|SL Ra, Rb|Same as the first, except it stores the whole Source Register B|
|Set Bit|SET Rx, k|Sets the "k"th bit in the specified Register, if the Register is R0 "k" is instead used as an 8-bit wide mask to select which bits in the SR should be set|
|Clear Bit|CLR Rx, k|Clears the "k"th bit in the specified Register, if the Register is R0 "k" is instead used as an 8-bit wide mask to select which bits in the SR should be cleared|
|Load Word Immediate (Signed)|LWI Re, k|Sign-extends the 16-bit constant "k" to 32-bits and loads that value into the Destination Register|
|Load Word Immediate (Unsigned)|LWIU Re, k|Same as above, except it zero-extends the 16-bit constant|
|Load Long Immediate|LLI Re, k|Same as the first, except the constant is 32-bit wide and therefore doesn't require "extending"|
|Return to 65816 Mode|EXIT|Exits the VM and resumes regular 65816 execution after the `sw32_execute` function|

some more detail about the Flags:<p>
the Zero Flag (Z) is set when the result of an operation is equal to 0<p>
the Negative FLag (N) is a copy of the MSB (bit 31) of the result of an operation<p>
the Carry Flag (C) depends on the operation:<p>
* Add/Add Immediate sets the Carry when the result overflowed (overflew?)
* Subtract sets the Carry when the result went below 0 (like an overflow but in the opposite direction)
* Shifts/Rotates set the Carry to the value of the bit that was shifted out, in addition Rotates shift  the previous Carry into the number as well

# 65816 Functions
There are 2 main functions required to make the VM Work, and 3 extra functions for printing
(ALL of them are called with JSL, and expect 8-bit A, and 16-bit X/Y)

`sw32_init` - clears all Registers, and sets the Control Byte to the value in A
Currently the Control byte's only used bit is bit 7, which determins the address width for Load/Store Instructions.
if bit 7 is cleared Load/Store Instructions are limited to 16-bit addressing, if set they are 24-bit instead.

`sw32_execute` - executes SW32 code, it starts at the address given by the X and Y Registers. X = Low Word, Y = High Word (High Byte is ignored)
The function only returns when an EXIT instruction is executed.

`sw32_print` - prints out the contents of all Registers in a nice and tidy format:
```
R1:  $00000000
R2:  $00000000
R3:  $00000000
R4:  $00000000
R5:  $00000000
R6:  $00000000
R7:  $00000000
R8:  $00000000
R9:  $00000000
R10: $00000000
R11: $00000000
R12: $00000000  PC: $00000000
R13: $00000000
R14: $00000000      7654NCZ
R15: $00000000  SR: 0000000
```
`sw32_print_h8` - Prints the 8-bit value in A

`sw32_print_h32` - Prints the 32-bit value that was pushed to the stack before calling the function (push the high word first)

in order for the print functions to work they needs a user provided function to print a single ASCII Character to whatever output the user might have
said function has to be called "sw32_print_char", return with RTL and has to assume A is 8-bits wide, and X/Y are 16-bits wide.

btw, There is no "sw32_print_h16" function, so if needed the user has to implement their own.
