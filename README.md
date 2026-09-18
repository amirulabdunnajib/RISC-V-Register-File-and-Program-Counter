# RISC-V Register File and Program Counter — Learning Notes

## Introduction

This repository documents my learning process as I build a multicycle RISC-V processor in Verilog. I am writing these notes both for my own revision and for others learning how processor components work together.

This section focuses on the **register file** and **program counter (PC)**: what they store, how they read and update data, and how they fit into a multicycle datapath. Later notes will explore the additional registers used to retain intermediate results and compare them with pipeline registers.

The repository is intended to include:

- Verilog source files, such as `design.v` (the current attached source is `RegisterFile_PC.v`).
- Explanations of the modules, signals, and datapath connections.
- A list of supported instructions and the operations required to execute them.
- Cycle-by-cycle notes for the multicycle implementation.
- A self-checking testbench, test cases, and simulation results as verification is developed.

**Status:** This is a work in progress. The instruction and test lists will be added later. The self-checking testbench is planned; these notes do not claim that the design has already passed verification.

## 1. Register File

The register file stores operands and results used by instructions. In this design, it contains **32 registers, each 32 bits wide**, named `x0` to `x31`.

For example, `add x5, x1, x2` reads the values stored in `x1` and `x2`, adds them in the ALU, and writes the result into `x5` when writeback is enabled.

| Instruction field | Bits | Purpose |
| --- | --- | --- |
| `rs1` | `[19:15]` | Select the first source register |
| `rs2` | `[24:20]` | Select the second source register |
| `rd` | `[11:7]` | Select the destination register |

These fields are used where applicable to the instruction format. Not every instruction uses both source registers or writes a destination register. Register numbers select entries in the register file; they are not memory addresses.

### Read and write behaviour

- **Two combinational read ports:** `o_Data_Rs_1` and `o_Data_Rs_2` reflect the selected register values after combinational propagation delay. Reading does not require a clock edge.
- **One synchronous write port:** on a rising clock edge, `i_Data_Rd` is written to `rd` when `i_Write_Enable = 1` and `rd != 0`, provided reset is inactive.
- **Zero register:** reset initializes `x0` to zero, and writes to `x0` are blocked.
- **Active-high asynchronous reset:** asserting `i_Rst` initializes the register bank without waiting for a clock edge.

The supplied RTL initializes `x2` (stack pointer) to `0x1001_03FC` with the default 1024-byte memory-size parameter, and `x3` (global pointer) to `0x1001_0000`. All other registers are reset to zero. These are project-specific initialization choices.

Having two read ports and one write port does **not** determine whether the processor is single-cycle, multicycle, or pipelined. That depends on the surrounding datapath and control logic.

## 2. Program Counter

The PC is a separate 32-bit register that holds the instruction address. It is not one of `x0`–`x31`.

In the supplied RTL:

- Reset sets the PC to `0x0040_0000`.
- `o_PC_Output` provides the stored PC value.
- `o_PC_Plus_4` combinationally produces `PC + 4`, the sequential address for 32-bit instructions.
- On each rising clock edge with reset inactive, the PC captures `i_Data`.

The top module selects the next PC as follows:

| `i_PC_Sel` | PC input |
| --- | --- |
| `0` | Current PC + 4 |
| `1` | `i_ALU_output` |

The writeback multiplexer selects the data supplied to the register file:

| `i_Write_Back_Sel` | Register-file write data |
| --- | --- |
| `00` | ALU output |
| `01` | PC + 4 |
| `10` | Memory data |
| `11` | Memory data |

The supplied file is a register-file/PC subsystem. It does not by itself contain a complete CPU or establish which instructions are supported.

## 3. What Happens in One Multicycle Clock Cycle?

In a non-overlapped multicycle processor, one instruction progresses through several cycles before the next instruction begins. During a cycle, combinational logic calculates values; at the active clock edge, enabled registers capture the values needed for later cycles.

The following is an **illustrative sequence**, not a confirmed schedule for the current RTL:

| Cycle or phase | Work during the cycle | Possible storage at the ending edge |
| --- | --- | --- |
| Fetch | Read instruction memory using the PC | Capture instruction in the instruction register (`IR`) |
| Decode | Decode the held instruction and read source registers | Capture operands in temporary registers, if used |
| Execute | Calculate an ALU result, effective address, or branch information | Capture results needed by later phases |
| Memory, when required | Perform a load or store | Capture returned load data when valid |
| Writeback, when required | Select the instruction result | Write the destination register |

The final FSM determines which phases are combined, skipped, or repeated. Memory latency can also affect the number of cycles.

**Multicycle adaptation needed:** the supplied PC has no write-enable input and currently updates every cycle. A multicycle design needs a controlled way to hold the PC, such as a PC write enable or a feedback selection. The instruction must also remain available across its execution, typically in an enabled `IR`. If the PC advances early, the original instruction PC or its `PC + 4` value must be retained for operations that need it later.

## 4. Temporary Registers and Pipeline Registers

Both store values across clock edges, but their role differs:

| Register type | Purpose | Examples |
| --- | --- | --- |
| Register file | Store instruction-visible operands and results | `x0`–`x31` |
| Multicycle temporary registers | Preserve intermediate values for the instruction being executed | `IR`, operand registers, ALU-result register, load-data register |
| Pipeline registers | Pass data and control between stages while different instructions overlap | `IF/ID`, `ID/EX`, `EX/MEM`, `MEM/WB` |

For example, an `ID/EX` pipeline register may hold source operand values, an immediate, an instruction PC, the destination register number, and control signals. A pipeline register is therefore often a **group of registers**, not just one 32-bit value. Pipeline control also needs ways to stall or invalidate entries.

Adding temporary registers to a multicycle CPU does not automatically make it pipelined. Pipelining requires overlapping the execution of different instructions.

## 5. Planned Verification

I plan to develop a **self-checking testbench** that applies stimulus, compares actual outputs with expected values, and reports pass/fail results automatically. Waveforms will support debugging when checks fail.

The detailed test list and expected results will be added later. Verification results will be documented after the testbench has been implemented and run.
