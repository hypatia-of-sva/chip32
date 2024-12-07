# Specification for the chip32 system.

## Overview

Chip32 is a 32-bit Computer in the following senses:

1. all registers are 32-bit, being usable for either 32-bit integer (signed and unsigned) or 32-bit IEEE floating point numbers (binary32), or for pointers.
2. pointers are 32-bit, that is, there are 2^32 = 4 billion addresses
3. data cells are 32-bit, that is, behind each address lie 4 bytes, so that the whole address space represents 4bytesx4billion= 16 GB.
4. all current instructions are a single cell, i.e. 32 bits long

There are 255 general-purpose registers with ids 0 to 254, and the program counter (PC) with id 255. This means any register is represented by a single byte.

There are also a number of flags, that are usable by special instructions; they are memory mapped, along with other important internal data, onto the last 256KB of memory, i.e. addresses with the upper 16 bits all 1. This area of memory is not usable as normal storage; writing and reading to it is handled specially, and trying to jump into this area will end the execution of the program, which is also used as the way of exiting program execution normally. The number and location of these flags and other memory-mapped data will be explained later on.

The representation of instructions in binary always has one of two forms:

1. opcode | Rn | imm
2. opcode | Rn |  Rm | Rl

opcodes are 8-bit, registers (Rn, Rm, Rl) also 8-bit, immediate values (imm) 16-bit. Not all opcodes are currently used, some are reserved for future use, including possibly multi-cell instructions that could have a different structure. However, the first 8 bits will always be used to represent the op code. In this representation, the first bit is the most significand (i.e. opcodes use the bits with values 2^31 to 2^28).

The currently supported instructions are split in the following sections, each explained with a pseudo-micro-code segment and a written out explanation:

## Instructions

### 1. Data transfer

#### Copy/Swap (op = 0)
CPSWP Rn, Rm, Rl:

	x = Rm; Rl = Rn; Rn = x;
 
This copies the value of Rm to Rn while copying the old value of Rn to Rl. If l == n, this is a copy from Rm to Rn; if l == m, this is a swap of Rm and Rn.

#### Load/Store (op = 1)
LDST Rn, Rm, Rl:

	if (Rl != 0) x = Rl*;
 	if (Rm != 0) Rm* = Rn;
  	if(Rl != 0) Rn = x;
 
This stores the old value of Rn to the cell pointed to by Rm, and loads the content of the cell pointed to by Rl. Null pointers are used to note that no saving or loading shall take place. If both are non-null, this is effectively a swap between register and memory cell. l == m is permitted, and it is guaranteed that the value loaded is the old value of the cell which is only overwritten after.

#### Load lower from immediate (op = 3)
LDLI Rn, imm

	Rn &= 0xFF00; Rn |= imm;
 
This sets the lower 16 bits of Rn to the immediate value, interpreted as a 16-bit unsigned integer.

#### Load upper from immediate (op = 4)
LDUI Rn, imm

	Rn &= 0x00FF; Rn |= (imm << 16);
 
This sets the upper 16 bits of Rn to the immediate value, interpreted as a 16-bit unsigned integer.

#### Load lower from immediate and zero (op = 5)
LDLIZ Rn, imm

	Rn = imm;

This sets the lower 16 bits of Rn to the immediate value, interpreted as a 16-bit unsigned integer, and zeroes the upper bits.

#### Load lower immediate and sign extend (op = 6)
LDLISE Rn, imm

	Rn = se(imm);

This sets Rn to the sign extended value of the immediate, interpreted as a 16-bit signed integer, as defined in a later section.

#### Load lower immediate from floating point (op = 7)
LDLIF Rn, imm

	Rn = ex(imm);

This sets Rn to the expanded version of the immediate, interpreted as a 16-bit floating point number (half-float), as defined in a later section.

#### op codes 8 to 15 are reserved for future use.

### 2. Integer arithmetic

#### Addition (op = 16)
ADD Rn, Rm, Rl: 

	Rn = Rm + Rl;
 
Adds two registers and saves the sum to a third in integer format. Uniform for signed and unsigned integers. Wraps around 2^32. Wrapping sets the carry flag.

#### Addition with carry (op = 17)
ADC Rn, Rm, Rl:

	Rn = Rm + Rl + carry
 
Adds two registers and the carry flag and saves the sum to a third register in integer format. Uniform for signed and unsigned integers. Wrapping behavior is the same as for the sequence of instructions:
	ADD Rn, Rm, Rl
	ADD Rn, Rn, carry
where carry is either 0 or 1 depending if the carry flag is set.

#### Subtraction (op = 18)
SUB Rn, Rm, Rl:

	Rn = Rm - Rl
 
Substracts two registers and saves the difference to a third in integer format. Uniform for signed and unsigned integers. Wrapping behavior is the same as for ADD when summing the values of Rm and of (2^32 - Rl), which is the same as the bit inverse of Rl plus 1. It acts therefore exactly as if executing

	NAND Rk, Rl, Rl
	ADDI Rk, #1
	ADD Rn, Rm, Rk
 
for any temporary register Rk, but without actually changing any register except Rn. Wrapping sets the borrow flag.

#### Subtraction with borrow (op = 19)
SBB Rn, Rm, Rl:

	Rn = Rm - Rl - borrow
 
Subtracts two registers and from that the borrow flag and saves the result to a third register in integer format. Uniform for signed and unsigned integers. Wrapping behavior is the same as for the sequence of instructions:

	SUB Rn, Rm, Rl
	SUB Rn, Rn, borrow
 
where borrow is either 0 or 1 depending if the borrow flag is set.

#### Multiplication (op = 20)
MUL Rn, Rm, Rl:

	Rn = Rm * Rl
 
Multiplies two registers and saves the product to a third in integer format. Uniform for signed and unsigned integers. Wraps around 3^32. The calculation performed is the unsigned calculation, which is equivalent to the signed calculation due to (2^32-k)(2^32-l) ^= kl mod 2^32 and k(2^32 - l) ^= 2^32-kl mod 2^32. Wrapping sets the overflow flag.

#### Unsigned Division (op = 21)
UDIV Rn, Rm, Rl

	Rn = Rm / Rl
 
Divides two registers and saves the whole number part of the quotient to a third in unsigned integer format. Both operands are taken as unsigned integers, hence the quotient is always positive and the whole number part, as less than Rm, always representable. If Rl == 0, Rn remains unchanged, and the divide-by-zero flag is set.

#### Signed Division (op = 22)
SDIV Rn, Rm, Rl

	Rn = Rm / Rl
 
Functions exactly as UDIV, including divide-by-zero flag, except that the operand are taken as signed integers, so the result is also stored as a signed integer.

#### Modulo (op = 23)
MOD Rn, Rm, Rl

	Rn = Rm % Rl
 
Takes the remainder of the quotient Rm / Rl, where both operands are interpreted to be unsigned integers, and stores it into a third register. The result will always be between 0 and Rl - 1. If Rl == 0, Rn remains unchanged and the divide-by-zero flag is set.

#### Bit-wise and (op = 24)
AND Rn, Rm, Rl

	Rn = Rm & Rl
 
Performs a bit-wise-and on two registers and saves the result in a third. Uniform for signed and unsigned integers. Flags are unchanged.

#### Bit-wise or (op = 25)
OR Rn, Rm, Rl

	Rn = Rm | Rl
 
Performs a bit-wise-or on two registers and saves the result in a third. Uniform for signed and unsigned integers. Flags are unchanged.

#### Bit-wise exclusive or (op = 26)
XOR Rn, Rm, Rl

	Rn = Rm ^ Rl
 
Performs a bit-wise-exclusive-or on two registers and saves the result in a third. Uniform for signed and unsigned integers. Flags are unchanged.

#### Bit-wise nand (op = 27)
NAND Rn, Rm, Rl

	Rn = ~(Rm & Rl)
 
Performs a bit-wise-nand on two registers and saves the result in a third. Uniform for signed and unsigned integers. Flags are unchanged.

#### Logical nand (op = 28)
LNAND Rn, Rm, Rl

	Rn = !(Rm && Rl)
 
Performs a logical nand on two registers and saves the result in a third. The result is either 0 or 1. Uniform for signed and unsigned integers. Flags are unchanged.

#### Left-shift (op = 29)
LSH Rn, Rm, Rl

	Rn = (Rm << Rl)
 
Performs a left-shift of one register by the number of bit-positions indicated in another, and saves the result in a third. Uniform for signed and unsignde integers, effectively calculating the result with all registers treated as unsigned integers. If the value of Rl is greater than 31, the overflow flag is set. Vacated bit-positions are filled with 0.

#### Unsigned right-shift (op = 30)
URSH Rn, Rm, Rl

	Rn = (Rm >> Rl)
 
Performs a right-shift of one register by the number of bit-positions indicated in another, and saves the result in a third, treating all registers as unsigned integers. If the value of Rl is greater than 31. the underflow flag is set. Vacted bit-positions are filled with 0.

#### Signed right-shift (op = 31)
SRSH Rn, Rm, Rl

	Rn = (Rm >> Rl)
 
Performs a right-shift of one register by the number of bit-positions indicated in another, and saves the result in a third, treating the the value shifted as a signed integer, the number of bit positions as an unsigned integer, and the result as a signed integer. If the value of Rl is greater than 31. the underflow flag is set. Vacted bit-positions are filled with the highest big, or sign-bit, of the original value that is being shifted.

#### Unsigned greater-than (op = 32)
UGT Rn, Rm, Rl

	if(Rm > Rl) Rn = 1;
	if(Rm <= Rl) Rn = 0;
 
Compares two registers as unsigned integers. Stores the result of a strict greater-than check in a third register. Also sets, depending on the relation between the two values, the equal, less than, or greater than flag. The value 1 is used so that multiplying with the result conditionally zeroes another value, so that comparisons can be used in arithmetic.

#### Signed greater-than (op = 33)
SGT Rn, Rm, Rl

	if(Rm > Rl) Rn = 1;
	if(Rm <= Rl) Rn = 0;
 
Compares two registers as signed integers. Stores the result of a strict greater-than check in a third register. Also sets, depending on the relation between the two values, the equal, less than, or greater than flag.

#### Equality comparison (op = 34)
EQ Rn, Rm, Rl

	if(Rm == Rl) Rn = 1;
	if(Rm != Rl) Rn = 0;
 
Compares two registers for equality. Stores the result in a third register. Does set the equal flag, but no other.


The following integer operations take immediate values. They are all to be treated as 16-bit _signed_ integers, i.e. they are first to be sign extended, by first copying the 16 expressed bits into the 16 lower bits of a 32-bit number, and then setting the upper 16 bits to either 0x0000 or 0xFFFF depending on if the hightest bit, or the 2^15-bit, in the immediate, was 0 or 1. Write se(imm) for the result of this sign-extension.

#### Addition with an immediate (op = 35)
ADDI Rn, imm:

	Rn += se(imm)
 
Works exactly like

	temp = se(imm)
	ADD Rn, Rn, temp
 
would, including all necessary flag changes.


In exactly the same way, with only the operation changed, we define the following immediate operations, in relation to the original op + I:


#### Addition with carry with an immediate (op = 36)
ADCI Rn, imm

#### Subtraction with an immediate (op = 37)
SUBI Rn, imm

#### Subtraction with borrow with an immediate (op = 38)
SBBI Rn, imm

#### Multiplication with an immediate (op = 39)
MULI Rn, imm

#### Unsigned Division with an immediate (op = 40)
UDIVI Rn, imm

#### Signed Division with an immediate (op = 41)
SDIVI Rn, imm

#### Modulo with an immediate (op = 42)
MODI Rn, imm

#### Bit-wise and with an immediate (op = 43)
ANDI Rn, imm

#### Bit-wise or with an immediate (op = 44)
ORI Rn, imm

#### Bit-wise exclusive or with an immediate (op = 45)
XORI Rn, imm

#### Left-shift with an immediate (op = 46)
LSHI Rn, imm

#### Unsigned right-shift with an immediate (op = 47)
URSHI Rn, imm

#### Signed right-shift with an immediate (op = 48)
SRSHI Rn, imm

#### Unsigned greater-than with an immediate (op = 49)
UGTI Rn, imm

#### Signed greater-than with an immediate (op = 50)
SGTI Rn, imm

#### Equality comparison with an immediate (op = 51)
EQI Rn, imm

There is no NANDI, because that would be the same as ORI with a bit-inverted immediate, and no LNANDI becsuse the operation, depending if the immediate is 0 or not, would be equivalent to either LNAND Rm, Rm, Rm or to EQ Rm, Rm, Rm.

Importantly, signedness of the operation does matter even though the immediate is always sign-extended. UGTI and SGTI act differently, because they treat the register as either signed or unsigned, and therefore, say if R5 has stored the value 0xFFFFF00, the operations

	UGTI R5, #17
	SGTI R5, #17
 
have opposite results, because the first instruction interprets R5 as a large positive integer, thus writing 1 to R5, while the second reads it as a negative integer, thus writing 0 to R5.

Also we add the following two immediate instructions:

#### Unsigned less-than with an immediate (op = 52)
ULTI Rn, imm:

Works exactly like

	temp = se(imm)
	UGT Rn, temp, Rn
 
would, including all necessary flag changes.

#### Signed less-than with an immediate (op = 53)
SLTI Rn, imm:

Works exactly like

	temp = se(imm)
	SGT Rn, temp, Rn
 
would, including all necessary flag changes.


The following instruction is a mixture of these two types of instructions:

#### Split integer register (op = 54)
SPLIT Rn, Rm, Rl

	Rm = se(Rn & 0xFFFF);
 	Rl = se((Rn & 0xFFFF0000) >> 16);

Takes apart a whole register into two parts, which are then sign-extended and stored into seperate registers.


#### op codes 55 to 79 are reserved for future use.


### 3. Floating-point arithmetic

#### Floating-point addition (op = 80)
FADD Rn, Rm, Rl	

	Rn = Rm + Rl
 
Computes the floating-point addition of two registers according to IEEE single-precision and stores it in a third.

#### Floating-point subtraction (op = 81)
FSUB Rn, Rm, Rl

	Rn = Rm - Rl
 
Computes the floating-point subtraction of two registers according to IEEE single-precision and stores it in a third.

#### Floating-point multiplication (op = 82)
FMUL Rn, Rm, Rl

	Rn = Rm * Rl
 
Computes the floating-point multiplication of two registers according to IEEE single-precision and stores it in a third.

#### Floating-point divison (op = 83)
FDIV Rn, Rm, Rl

	Rn = Rm / Rl
 
Computes the floating-point division of two registers according to IEEE single-precision and stores it in a third.

#### Floating-point modulo (op = 84)
FMOD Rn, Rm, Rl

	Rn = Rm % Rl
 
Computes the floating-point modulo of two registers and stroes it into a third. This means, that the result is the same as the value Rm - k * Rl for a (positive or negative) integer k, such that the result has the same sign as Rm and is closer to zero than Rl. If Rl == 0, a floating-point divide-by-zero error is set. If any of the arguments are NaN or infinity, a floating-point error is set.

#### Floating-point exponentiation (op = 85)
FEXP Rn, Rm, Rl

	Rn = Rm ** Rl
 
Computes the floating-point exponentiation of two registers and stores it in a third. For Rl == 0.5f, the result is calculated according to the IEEE sqrt-operation, performed on the value of Rm. Otherwise: If Rm < 0 and Rl is not a whole number, or if Rm == 0 and Rl <= 0, or if any of the arguments are NaN or infinity, a floating-point error is set. If the result is not representable due to size, Rn will be set to +infinity or -infinity depending on the sign of the actual result. If the result is not representable due to being to close to zero, Rn will be set to +0 or -0 depending on the actual result, and the underflow-flag will be set.

#### Floating-point logarithm (op = 86)
FLOG Rn, Rm, Rl

	Rn = log Rl (Rm)
 
Computes the logarithm to the base of the value of one register of another, both interpreted as floating-point numbers, and stores it in a third. If Rm <= 0, or if any of the arguments are NaN or infinity, a floating-point error is set. If the result is not representable due to size, Rn will be set to +infinity or -infinity depending on the sign of the actual result. If the result is not representable due to being to close to zero, Rn will be set to +0 or -0 depending on the actual result, and the underflow-flag will be set. 

#### Floating-point sine (op = 87)
FSIN Rn, Rm, Rl

	Rn = Rl * sin (Rm)
 
Computes the product of one register with the sine of another and stores it to a third. If any of the arguments are NaN or infinity, a floating-point error is set. If the result is not representable due to being to close to zero, Rn will be set to +0 or -0 depending on the actual result, and the underflow-flag will be set.

#### Floating-point inverse sine (op = 88)
FASIN Rn, Rm, Rl

	Rn = arcsin(Rm / Rl)
 
Calculates the principal value of the inverse sine of one register divided by another and stores it to a third. If Rl == 0, a divide-by-zero floating-point error is set. If the value of the quotient is not in the range from -1 to +1, or if any of the arguments are NaN or infinity, a floating point error is set. If the result is not representable due to being to close to zero, Rn will be set to +0 or -0 depending on the actual result, and the underflow-flag will be set.

#### Floating-point greater-than (op = 89)
FGT Rn, Rm, Rl

	if(Rm > Rl) Rn = 1;
	if(Rm <= Rl) Rn = 0;
 
Compares two registers as single precision floats. Stores the result of a strict greater-than check in a third register. Also sets, depending on the relation between the two values, the equal, less than, or greater than flag. If any of the arguments are NaN, a floating point error is set.

#### Floating-point equality comparison (op = 90)
FEQ Rn, Rm, Rl

	if(Rm == Rl) Rn = 1;
	if(Rm != Rl) Rn = 0;
 
Compares two registers for equality as floating point numbers. Stores the result in a third register. Does set the equal flag, but not the less than or greater than flag. NaNs are the only values that compare unequal to themselves (however, they do compare equal to themselves under the EQ instruction; this is important e.g. to implement functions like fpclassify or isnan).

#### Convert integer into floating-point number (op = 91)
I2F Rn, Rm, Rl

	Rn = Rm * (2**Rl)
 
Takes two registers as significand and exponent of a new floating point number, which is then shifted to the corrext exponent for significance, reduced of the leading 1, and stored into a third register. If the result is not representable due to being to close to zero, Rn will be set to +0 or -0 depending on the actual result, and the underflow-flag will be set.

#### Round floating-point number to integer (op = 92)
F2I Rn, Rm, Rl

	Rn = round(Rl)
	Rm = Rl - round(Rl)
 
Rounds one register according to the current rounding mode (stored in a flag at a special address), and stores the result and the remainder into two other registers. If any of the arguments are NaN or infinities, a floating point error is set. If the result is not representable due to size, Rn will not be set and the overflow flag will be set; however, Rm will still be set. The original value Rl and the remainder Rm are here treated as a floating-point registers, while Rn is treated as a signed integer register.


The following floating-point operations take immediate values. They are all to be treated as 16-bit IEEE floating-point numbers, i.e. in the format Binary16, also called half-floats. They are first to be extended into 32-bit floats, by the following procedure: The expenent is the sign-extended version of the old exponent. The mantissa is set to be equal to the old data in the _higher_ bits, with the lower bits being filled with 0s. The sign bit is copied. This means, for example, that the 16-bit float 0b1|01101|0110111001 is extended to 0b1|00001101|01101110010000000000000. Write ex(imm) for this extension.


#### Floating-point addition with an immediate (op = 93)
FADDI Rn, imm:

	Rn += ex(imm)
 
Works exactly like

	temp = ex(imm)
	FADD Rn, Rn, temp
 
would, including all necessary flag changes.


In exactly the same way, with only the operation changed, we define the following immediate operations, in relation to the original op + I:


#### Floating-point subtraction with an immediate (op = 94)
FSUBI Rn, imm

#### Floating-point multiplication with an immediate (op = 95)
FMULI Rn, imm

#### Floating-point divison with an immediate (op = 96)
FDIVI Rn, imm

#### Floating-point modulo with an immediate (op = 97)
FMODI Rn, imm

#### Floating-point exponentiation with an immediate (op = 98)
FEXPI Rn, imm

#### Floating-point logarithm with an immediate (op = 99)
FLOGI Rn, imm

#### Floating-point sine with an immediate (op = 100)
FSINI Rn, imm

#### Floating-point inverse sine with an immediate (op = 101)
FASINI Rn, imm

#### Floating-point greater-than with an immediate (op = 102)
FGTI Rn, imm

#### Floating-point equality comparison with an immediate (op = 103)
FEQI Rn, imm


Also we add the following immediate instructions:

#### Floating-point less-than with an immediate (op = 104)
FLTI Rn, imm:

Works exactly like

	temp = ex(imm)
	FGT Rn, temp, Rn
 
would, including all necessary flag changes.


#### Convert immediate integer into floating-point number (op = 105)
II2F Rn, imm

Works exactly like:

	temp = se(imm)
	I2F Rn, temp, 1
 
would, including all necessary flag changes.

#### Round immediate floating-point number to integer (op = 106)
IF2I Rn, imm

Works exactly like	

	temp = ex(imm)
	F2I Rn, ignore, temp
 
would, including all necessary flag changes.


The following instruction is a mixture of these two types of instructions:

#### Splt floating-point register (op = 107)
FSPLIT Rn, Rm, Rl

	Rm = ex(Rn & 0xFFFF);
 	Rl = ex((Rn & 0xFFFF0000) >> 16);

Takes apart a whole register into two parts, which are then interpreted as 16-bit floats, extended to 32-bit floats, and stored into seperate registers.


#### op codes 108 to 120 are reserved for future use.



### 4. Control-flow instructions and miscelaneous

#### Branch if zero (op = 121)
BZ Rn, imm

	if(Rn == 0) PC += se(imm);
 
Adds an offset to the program counter, if a register is zero. This offset is interpreted as a 16-bit signed integer, thus sign extended, so this instruction can jump backwards and forwards 32k instructions.

#### Interrupt invocation (op = 122)
INT Rn, imm

	(interrupt_table[Rn])(imm)
 
Invokes the interrupt with the number of the value in Rn, with some immediate value as payload. How this payload is to be interpreted - whether as numbers, addresses to certain pages, or as two register numbers, or in other ways - is up to the interrupt handler / built-in functionality, and documented there. These interrupts can be hardware functionality (like making a controller rumble), a chip32 system functionality (like saving savegames) or a user supplied interrupt handler (like shader code for draw calls). The instruction can take an indeterminate amount of time and may or may not return immediately while the task is completed asynchronously in the background (for example, saving should happen immediately and be blocking, but draw calls, as they are potentially executed on an external GPU, will be executed asynchronously).

#### op codes 123 to 255 are reserved for future use

### Conclusion

There are in total 76 of 256 opcodes in use. The other 180, that is 8-15, 55-79, 108-120 and 123-255, are reserved for data transfer, integer, floating point and miscelaneous instructions respectively, and set the illlegal instruction flag in its current use. If activated, they also call an illegal instruction trap handler. If not, they will act as nops. However, for intentional nops there are legal instructions, such as CPSWP Rn, Rn, Rn or ADDI Rn, #0, that should be used instead, as the behavior of any program using illegal instructions cannot be guaranteed in case future versions take use of the now reserved opcodes.






## Memory-mapped utilities

### a) CPU internal-flags.

There are a number of internal flags and settings accessible to the user. Instructions can set the following flags:

1. carry
2. borrow
3. overflow
4. underflow
5. divide by zero
6. floating-point error (invalid operation)
7. floating-point inexactness (in last operation)
8. illegal instruction
9. termination (if jumped to addresses beginning with 0xFFFF)

The IEEE divide-by-zero, overflow and underflow are mapped to the corresponding flags, as well as other instructions (also integer instructions). Carry and borrow also always set the overflow flag.

The flags are stored, in the order from least to highest significant bit, at the address 0xFFFF0000, so currently using bits from position 1 to 2^8. Reading this address will return the flags, writing to it is an illegal instruction and raises the illegal instruction flag.

For each flag, it can be set whether or not the trap handler should be called when it gets set. This information is of the same layout as the flags and stored at the address 0xFFFF0001. It can be read, and it can be written to, as long as only used flag bits are set to 1. If any other bits are set to 1, writing to it is an illegal instruction and raises the illegal instruction error.

The current floating-point rounding mode is stored in the least significant two bits of 0xFFFF0002 and is one of the following:
- 0: round to zero
- 1: round towards positive infinity
- 2: round towards negative infinity
- 3: round to nearest (tie is not clearly defined)
It can be read, and one of the allowed values can be written to. If any except the least significant two bits are changed, writing to it is an illegal instruction and raises the illegal instruction error. It will change the rounding for floating-point instructions as defined in IEEE.

The addresses 0xFFFF0003 to 0xFFFF00FF are reserved for future use.


### b) CPU-side interrupts.

There are a number of interrupts that the CPU might call when something happens. These are not user-callable by the INT instruction, and instead are only called by the CPU itself. These are the following:

- (at 0xFFFF0100) Trap handler for any of the flags that are on to monitor
- (at 0xFFFF0101) key, joystick and mouse input
- (at 0xFFFF0102) incoming package from network
- (at 0xFFFF0103) sound input (microphone)
- (at 0xFFFF0104 and 0xFFFF0105) vertex and fragment shader

For each of these interrupts, we can set a handler, by setting the mempory cels at addresses 0xFFFF0100 to 0xFFFF01FF. If a specific interrupt ought to not be handled, the cell can be set to 0, the null pointer. For all handlers, the code starts from the written address and must end at a LDLIZ R55, 0xFFFF. PC can't be written to directly in interrupt routine otherwise, and the BEZ instruction must only have immediate offsets that don't exit that range. All handlers except shaders are directly executed on the event. However, since shader code is not executed on the CPU, it must be compiled first. In order to aid that, there are the special addresss 0xFFFF0200-0xFFFF027F and 0xFFFF0280-0xFFFF02FF to keep 128 vertex resp. fragment shaders on hold.

When setting a handler like this, the CPU locks the associated memory, making a write to it an illegal instruction. Also, in the case of shaders, the write to the shader interrupt pointer or to one of the on-hold positions may take an indeterminate amount of time.

These handlers take in their input in registers. Shaders cannot use main memory, others can and should. If they have an output, it must be put in registers also.

The details on how these handlers are implemented will be provided in the corresponding descriptions below.

### c) System status information

A number of system status information can also be read from mapped memory, namely from addresses 0xFFFF0300-0xFFFF03FF. They are read-only, writing to it is an illegal instruction and raises the illegal instruction error. The following are currently defined, the rest are reserved for future use:

- 0xFFFF0300: uptime of the program in nanoseconds
- 0xFFFF0301: current time, with seconds in the lowest 8 bit, minutes in the next 8 bits, hours in the next 8 bits and a time zone code in the higest 8 bits, with the most significant bit signifying DST.
- 0xFFFF0302: current date, with days in lowest 8 bits, months in the next 8 bits, and the year in the highest 16 bits, stored as a signed number for BC years, in case the system is used by an active time traveller.
- 0xFFFF0303: current locale, with the lower 16 bits representing language, the next 8 bits country, and the highest 8 bits for currency
- 0xFFFF0304: an estimate of the current network connectivity in bit/second

