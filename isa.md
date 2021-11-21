# RISC-V ISA

This article gives you a quick tour to RISC-V ISA format and slots so that
the basic sense and implementation of extending the RISC-V ISA are covered.

These external links are involved, you can open them in your browser tabs in advance:
 * [The RISC-V Instruction Set Manual](https://riscv.org/wp-content/uploads/2019/12/riscv-spec-20191213.pdf)
 * [risc-v opcodes](https://github.com/riscv/riscv-opcodes)
 * [RISC-V GNU Compiler Toolchain](https://github.com/riscv/riscv-gnu-toolchain)

## Instruction Format

All the RISCV instructions are 32-bit vectors.
Please refer page 130 (# on the upper right of each page, not PDF reader page) of the
[RISC-V manual](https://riscv.org/wp-content/uploads/2019/12/riscv-spec-20191213.pdf)
for the format of these vectors.

To choose a proper format for your extended instruction, the number of src/dst
operands, the type of operands (imm/reg), and the number of bits to be encoded
should be considered. R, I, and S are recommended --- B is just a complicated
version of S, and U and J have no `funct3` field, which may occupy the whole
line of instruction slots (refer to [next section](#constraints)
for more detials). Therefore, we have 32 bits in total: 7 bits are  occupied
by the opcode; 3 bits are occupied for `funct3`; each register occupies 5 bits
($2^5=32$ ISA registers); immediate operand can either be 7 or 11 bits.

The instruction format are described in the [risc-v opcodes](https://github.com/riscv/riscv-opcodes)
repo, and you can open `opcodes-rv32i`, the most basic module of the RISC-V ISA,
for examples. To understand this file, we use
[addi](https://github.com/riscv/riscv-opcodes/blob/03be826f17faedcaee7f60223f402850e254df0a/opcodes-rv32i#L24)
instruction as an example, and a correspondence to the `ADDI` row in page 130 can be made.
Both the figure and the text description are little endian format.

````
addi    rd rs1 imm12           14..12=7 6..2=0x04 1..0=3
````

`rd`, `rs1`, and `imm12` describe the operands of this instruction; `14..12` describes `funct3`;
`6..2` describes the opcode. According to Table 24.1 on page 129 (no page number on the PDF),
the first two bits are always `11`.

For more information on the operand tokens appear in this file, refer to
[this](https://github.com/riscv/riscv-opcodes/blob/03be826f17faedcaee7f60223f402850e254df0a/parse_opcodes#L17-L49)
for more details. This Python dict declares the bit range this token occupies.
The semantics of each token id can be understood by knowing their bit range,
acompanied with the figure of the instruction format.

## Constraints

To understand the constraints of extending new instructions, we need to know:
 1. where are the available slots for the extended instructions, and
 2. some additional rules/standards/contraints of each slot.

Refer custom-0/1/2/3 cells in Table 24.1 on page 129. These four slots are reserved
for instruction extension.

Refer [this file](https://github.com/riscv/riscv-opcodes/blob/master/opcodes-custom)
for the operand constraints of each instruction. The operand signature of each instruction should
be exactly the same as their corresponding slot in custom.

**Do read this!! If you do not want to refactor the ISA aggresively when already having a large project!!**
You **cannot** give `funct3` random values. The meanings of the 3 bits of `funct3` are critical:
 * 1: Send `rs2` from host to the accelerator.
 * 2: Send `rs1` from host to the accelerator.
 * 4: Receive a value from the accelerator to `rd`.

`rs2` only appears when `rs1` appears. Therefore, the bit of `1` cannot be enabled alone. Therefore,
`1` and `5` cannot be `funct3`. Also, as mentioned above, if we want to use `0` as `funct3`, we cannot
use U-type format.

## Assembler Integration

After designing how instructions look like in your mind, we need to integrate them to the compiler, both the
**binary encoding** and the **text mnemonic**. This is done by hacking the subrepo,
`riscv-gnu-toolchain/riscv-binutils`.

### Binary Encoding

To integrate the binary encoding of the extended instruction, we want to replace the code segments
([1](https://github.com/riscv/riscv-binutils-gdb/blob/2cb5c79dad39dd438fb0f7372ac04cf5aa2a7db7/include/opcode/riscv-opc.h#L550-L597),
[2](https://github.com/riscv/riscv-binutils-gdb/blob/2cb5c79dad39dd438fb0f7372ac04cf5aa2a7db7/include/opcode/riscv-opc.h#L1106-L1129))
related to customized opcodes by the extended encoding.

In [risc-v opcodes](https://github.com/riscv/riscv-opcodes), scripts are provided to generate these encoding
codes. Use the following command:

```bash
cat opcodes-custom | ./parse-opcode -c > snippet
```

Edit `opcodes-custom` to name the extended instructions, and define the operands.

Not every line of `snippet` is useful, open the file and find the corresponding lines.

Copy those lines and use them to replace the code segments mentioned above.

### Mnemonic Format

To integrate the mnemonic (text) format of the extended instruction, we want to add additional rules below
[this line](https://github.com/riscv/riscv-binutils-gdb/blob/2cb5c79dad39dd438fb0f7372ac04cf5aa2a7db7/opcodes/riscv-opc.c#L199). The meaning of each column is:
* Name string;
* The default data width; zero means the same as machine bits; here I suggest to give 0;
* The module of the instruction belongs to; here I suggest just give "I", the most basic module;
* The operand description; there is no document for the meaning of each letter, but you can refer to
  [this git issue](https://github.com/riscv/riscv-binutils-gdb/issues/243) and read the source code for more 
  details; typically, knowing `s`, `t`, `j`, `d`, and `q` are enough;
* For instructions without aliasing and pesudo representation, the next two columns can just give the `MASK_*` 
  and `MATCH_*` generated in `snippet`.
* I believe it should be something about the aliasing and pseudo thing too, and giving `0` should also suffice.

### Implementation

This section includes some design descision I made. Though subjective, I hope this may more or less help your
development experience. I adopt an [auto-patcher](https://github.com/PolyArch/dsa-riscv-ext/) in my project.
This patcher should be in the same folder as `riscv-gnu-toolchain`. Say you have a directory `stack/`[^1], then
both `riscv-gnu-toolchain` and `patcher` should be in `stack/`.
I made this design decision for the following reason:
1. I want to minimize the invasion to the gnu toolchain so that the cost of rebasing gnu toolchain
   can also be minimized;
2. `opcode-custom` and `riscv-opc.c` should be updated together to comply to the same format of the extended
   instructions, so it is highly desirable to have a unified programming interface to generate and integrate
   both. For further development, this interface can also be used to generate ISA extension to LLVM toolchain.
3. Last but not the least, it involves many manual "select, copy, and paste" stuff, which is error-prone and
   breaks the automaticity of the compilation flow of the infrastructure stack.

Refer to [isa.ext](https://github.com/PolyArch/dsa-riscv-ext/blob/master/isa.ext), I have a text format to
describe how the extended instructions look like. Then refer to the
[Makefile](https://github.com/PolyArch/dsa-riscv-ext/blob/master/Makefile) and
[auto-patch.py](https://github.com/PolyArch/dsa-riscv-ext/blob/master/auto-patch.py)
for how the involved files are modified to integrate the extended instructions.

[^1]: For the structure of the `stack` directory, refer to [this repo](https://github.com/polyarch/dsa-framework);