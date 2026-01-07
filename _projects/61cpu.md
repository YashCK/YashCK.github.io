---
hasThumbnail: false

---

## Background

This was a course project focused on building a CPU that cana run RISC-V instructions using Logisim.

- **ALU:** built a configurable ALU that supports common integer ops (add/sub, bitwise ops, shifts, compares) plus a few multiply variants
- **Register file:** implemented a 32-register file with two read ports and one write port, with x0 hardwired to 0.
- **Immediate generator:** added logic to produce immediates for the different RISC-V instruction formats

Started with a tiny CPU that only ran `addi`, then expanded the datapath and control step-by-step to support more instruction types (selecting ALU inputs, choosing the next PC, and picking the right writeback value).

The CPU supported 
- **Integer ALU instructions** (R-type and I-type)
- **Branches**
- **Loads and stores**
- **Jumps and upper-immediate instructions** (`jal/jalr`, `lui/auipc`)

The CPU also was pipelined into 3-stages and handled the main correctness issue of **control hazards**. When a branch/jump changes the PC, the CPU flushes the wrong-path instruction by inserting a no-op (`addi x0, x0, 0`).
