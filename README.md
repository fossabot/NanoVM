# NanoVM
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Frajbhushanm100%2FNanoVM.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Frajbhushanm100%2FNanoVM?ref=badge_shield)

PoC lightweight x64 VM implementation

### Table of contents

- [NanoVM](#nanovm)
  * [General](#general)
  * [How to build](#how-to-build)
    + [Windows (Visual Studio 2019)](#windows--visual-studio-2019-)
    + [Debian](#debian)
  * [VM architecture](#vm-architecture)
    + [Registers](#registers)
    + [Instructions](#instructions)
- [NanoAssembler](#nanoassembler)
- [NanoDebugger](#nanodebugger)

## General 

NanoVM is cross-platform register based turing complete VM with stack memory. The project also includes assembler and debugger with similiar syntax to x86 asm with intel syntax. 
Note that the project is still in very early development and many things including the insturction set and format is a subject to change, so bytecode from previous versions might not work in future. 
The documentation will be updated when changes happen.

The longer term goal of the project is to be embeddable VM with a small bytecode format while maintaining reasonable performance speed. 
Syscall instruction that contains some implemented functions like IO but user can register custom functions as callbacks for different syscall function values will be added eventually when the bytecode format has been finalized. 
This allows one to implement more "outside of the VM" functionality". Performance comparison tests to other languages will be added later. 
Longer term goal is to eventually actually program the Compiler/Assembler in NanoVM bytecode. 

Note that even though the VM does do bounds checking for read write and execute operations on memory these checks are more for catching bugs in the code + avoiding VM crashing, and not so much about hardening the VM. 
Escaping the VM sandbox is likely very trivial. 
However, if you notice a way to read, write or execute memory outside of the VM I'll gladly fix those. 
That being said **!this VM should not be used to run unknown and potentially hostile code!**. 
Also stuff like executing stack memory is currently possible and this is made on purpose to allow dynamic code generation or encryption. 
Read/write/execute permissions to memory pages might be added in future.

## How to build

Build instructions have been tested on Windows and Debian based linux distros

### Windows (Visual Studio 2019)

You need to have Visual Studio 2019 and cmake installed on your system.\
Visual Studio 2019 is compatible with cmake projects so you can build the project by opening the project in visual studio, right click the root CMakeLists.txt -> "Generate Cache for NanoVM". This will generate the cmake cache for you and now you can build the project by selecting 
from the menu bar: Build -> Build all.\
If you rather wish to generate visual studio specific build files you can do that by running the following command in the project root with cmd/powershell:

```
cmake . -B ./build
```

This will generate new Visual Studio build files under build/

### Debian

You need to have build tools and cmake available. You can install those by running the following commands in terminal 
```
sudo apt install build-essentials
sudo apt install cmake
```
Now to build the project run the following commands
```
git clone https://github.com/etsubu/NanoVM.git
cd NanoVM
cmake .
make
```
This will build all the binaries in their own folders along the source files.

## VM architecture

The VM memory are defined as pages which by default are 4096 bytes each. When initialized the VM bytecode will be placed at the bottom of the allocated memory followed by the stack memory base on the next page. While the VM is similiar to x86 the stack grows up unlike in x86. This can be utilized to dynamically increase the stack memory if required with minimal effort.

### Registers
The VM is register based so the instuctions utilize different registers. Registers are encoded with 3 bits so there are 8 registers in total (the names will change in future):

| Register        | Number        | Description                                  |
| -------------   |:-------------:| --------------------------------------------:|
| Reg0            | 0             | General purpose. Used to store return values |
| Reg1            | 1             | General purpose.                             |
| Reg2            | 2             | General purpose.                             |
| Reg3            | 3             | General purpose.                             |
| Reg4            | 4             | General purpose.                             |
| Reg5            | 5             | General purpose.                             |
| Reg6            | 6             | General purpose.                             |
| Esp             | 7             | Stack pointer. Points to the top of the stack|

### Instructions
Instructions have always an opcode and 0-2 operands. Below is the instruction encoding defined from LSB to MSB

| 5 bits           | 3 bits                | 1 bit             | 2 bits                      | 1 bits        | 1 bit         | 3 bits        |
| -------------    |:---------------------:|:-----------------:|:---------------------------:|:-------------:|:-------------:|:-------------:|
| Opcode           | Destination register  | Source type       | Source size                 | Is_Dst_pointer| Is_Src_pointer|Source register|
| What instruction | Update this register  | Reg=0, Immediate=1| Byte, short, dword, qword   | True,false    | True, false   | Source register if src type is reg|

So most of the instructions are encoded in 2 bytes + immediate value if used. Instructions that use zero operands effectively being only 1 byte are:
```assembly
Halt ; Stops the execution and exits the VM execution
ret ; Pops value from the top of the stack and performs absolute jump to that address. Updates stack pointer
```
Instructions that use 1 operand do not use either source register or immediate value. They do not use destination register even though it is always defined. Opcodes that use 1 operand:
```assembly
	Jz; Jump if zero flag is set. Example: jz reg0
	Jnz; Jump if zero flag is not set. Example: jnz reg0
	Jg;  Jump if greater flag is set. Example: jg reg0
	Js;  Jump if smaller flag is set. Example: js reg0
	Jmp; Jump ("goto") instruction. Example: jmp reg0
	Not; Flip the bits in value. Example: not reg0
	Inc; Increases the value by one: Example inc reg0
	Dec; Decreases the value by one: Example dec reg0
	Call; Pushes the next instructions absolute memory address to the stack and performs relative jump to the given address. Updates stack pointer Example: call reg0
	Push; Pushes value to the top of the stack. Example: push reg0
	Pop; Pops value from the top of the stack and moves the value to given address. Example: pop reg0
	Printi; prints given integer. Example: printi reg0
	Prints; prints given null terminated string. Example: prints @reg0 | Note that @reg0 uses reg0 as pointer to the string not as an absolute value
	Printc; prints given ASCII char to the console. Example printc reg0
```
Instructions with 2 operands:
```assembly
	Mov; mov reg0, reg0 <=> reg0 = reg0
	Add; add reg0, reg0 <=> reg0 += reg0
	Sub; mov reg0, reg0 <=> reg0 -= reg0
	And; mov reg0, reg0 <=> reg0 &= reg0
	Or;  or reg0, reg0 <=> reg0 |= reg0
	Xor; xor reg0, reg0 <=> reg0 ^= reg0
	Sar; sar reg0, reg0 <=> reg0 >>= reg0
	Sal; sal reg0, reg0 <=> reg0 <<= reg0
	Ror; ror reg0, reg0 <=> performs circular shift to the right on reg0, by reg0 times
	Rol; rol reg0, reg0 <=> performs circular shift to the left on reg0, by reg0 times
	Mul; mul reg0, reg0 <=> reg0 *= reg0
	Div; div reg0, reg0 <=> reg0 /= reg0
	Mod; mod reg0, reg0 <=> reg0 %= reg0
	Cmp; cmp reg0, reg1 | Compares the 2 values and sets flags depending on the comparison.
```
ToDo:
* Remove print instructions and move them under the syscall instruction to operate with stream pointers. This allows the printing to support console IO and for example file IO
* Implement syscall instruction

# NanoAssembler
NanoAssembler is currently a minimalistic assembler for NanoVM. The assembler was made to aid in making simple programs and tests. This project is not so much about making a "programming language" but rather the core VM which could be used as the base which some programming language is compiled to. When more advanced features will be introduced I'll consider creating a new compiler project and leave the assembler for the low level operations.
Currently the assembler supports comments with prefix ';' and uses regex to filter multiple whitespaces to help in processing the input. The assembler also suppors labels which are defined by ':' prefix. This will be mapped to a memory address that points to the next instruction after label. Example:
```assembly
; The assembler supports comments
; The assembler strips multiple whitespaces
;          xor        reg0,     reg1 
; The above line would be translated to the one below. So the assembler is not sensitive with whitespaces
xor reg0, reg0 ; zero out reg0
:label
printi reg0 ; Label points here
printc '\n' ; The assembler can map characters defined with '' and special characters line \n \r \t to their ascii values
; The above line is the same as printc 10
inc reg0 ; reg0++
cmp reg0, 0x10 ; compare reg0 to 0x10 in hex which is the same as cmp reg0, 10
; The assembler understands base10 and base16 values
jnz label    ; if reg0 != 10 jump to label
; The above code will print numbers
```
ToDo:
* Add macros. These would help to reduce the amount of code that needs to be written.
* Add include tags which would allow to write "standard libraries" which could be included to the project
* Size definitions for registers
* ...

The assembler projects code is not currently clean and the development for that will be most likely be stopped eventually and a new compiler project will be started. Probably with external library for parsing the programming language. I will try and keep the assembler simple

# NanoDebugger

The project contains also a simple command line debugger + disassembler. The debugger inherits the NanoVM core and is capable of stepping through the programs. It also supports:
* Breakpoints
* Goto. This allows you to change the current instruction pointer
* print registers. This will print the current register values and flags set by cmp
* Print stack. This will print the stack memory up to the stack pointer. Each line of the dump will be 8 hex values followed by the same values in ascii separated by |. This allows to easily look at potential ASCII strings in stack as well as 64bit integers.
Todo:
* Add commands for modifying the stack and registers
* Add whole memory dump which will dump all the memory pages including code and stack to the disk.
* Add option to disassemble the whole code and dump to the disk with memory offsets


## License
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Frajbhushanm100%2FNanoVM.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Frajbhushanm100%2FNanoVM?ref=badge_large)