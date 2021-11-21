# Being a Full-Stack Hacker: From ISA to Silicon

The knowledge of hacking the full-stack software/hardware infrastructure
is very fragmented, which are either some questions on Stack Overflow or
learnt from senior colleagues orally. At worst, maybe reading the codebase
is necessary. I am writing these blogs for to save some of your time (hopefully)
to gather these fragments in a "whole body". These blog chapters cover:
1. hints on finding the files you potentially want to look at and modify to
   extend the functionality;
2. engineering tricks to improve the experience of development;
3. experiences to avoid the pitfalls in the existing codebase;
4. undocumented details to make you better understand how the origin
   project works.

To examplify the extension, these blogs will use my research infrasture,
[DSAGEN](https://github.com/polyarch/dsa-framework), as an paradigm of hacking.
Though my project is used for explanation, I will try to be as goal-neutral
as possible, so that these experiences can be reused.

**Note:** Unlike a paper publication, which takes infinte iterations to edit,
to avoid careless mistakes and polish language, these blogs inevitably have some
flaw. If you have any suggestions, do not hesitate to contact and suggest me for
updates.

## Contents

These blogs is stuck to the [RISC-V ISA](https://riscv.org/). If after finishing
this I find these blogs interesting, there will be x86 version too.

* RISCV ISA
  * [Extending the ISA](./isa.md)
  * [Programming in Intrinsics](./intrin.md)
* Simulating the Extended ISA in Gem5
  * [Decoding and Dispatching the Extended ISA](./decode.md)
  * General-Purpose Pipeline
* LLVM
  * Extending the ISA
  * Hacking the Parser
  * Writing a Transformation Pass
* Chisel
  * TBD
* Chipyard
  * TBD
