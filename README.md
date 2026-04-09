# RISC-V Single-Cycle CPU — 64-bit Verilog Implementation

**Course:** CSCI 311 — Computer Architecture  
**Institution:** Nile University, Cairo, Egypt — ITCS Department  
**Team:** Marwan Negm · Ahmed Afifi · Steven Wilson · Abdullah El-Dakhakhny · Mark Hany

---

## Project Overview

A fully functional 64-bit single-cycle RISC-V CPU implemented in Verilog HDL. The processor decodes and executes instructions in a single clock cycle, following the standard RISC-V single-cycle datapath. All modules were developed independently, verified with dedicated testbenches, and then integrated into a top-level CPU module.

---

## File Structure

```
RISC-V Processor/
│
├── Sources
│   ├── Abdullah_231000038_Task_1.v       # ALU
│   ├── Marwan_231000222_Task_2.v         # Control Unit (C_Unit)
│   ├── Ahmed Afifi_231000402_Task_3a.v   # Register File
│   ├── Ahmed Afifi_231000402_Task_3b.v   # Immediate Generator (ImmGen)
│   ├── Steven_231001331_Task_4.v         # Program Counter
│   ├── Mark_231000774_Task_5a.v          # Instruction Memory
│   ├── Mark_231000774_Task_5b.v          # Data Memory
│   ├── Mark_231000774_Task_5c.v          # Top-Level CPU (RISKV_CPU)
│   └── instructions.mem                  # Test program (14 instructions, hex)
│
└── Testbenches
    ├── Abdullah_231000038_Task_1.v        # ALU testbench (testing_ALU)
    ├── Marwan_231000222_Task_2.v          # Control Unit testbench (C_Unit_tb)
    ├── Ahmed Afifi_231000402_Task_3a.v    # Register File testbench (RegisterFile_tb)
    ├── Ahmed Afifi_231000402_Task_3b.v    # Immediate Generator testbench (ImmGen_tb)
    ├── Steven_231001331_Task_4.v          # Program Counter testbench (tb_ProgramCounter)
    └── Mark_231000774_Task_5.v            # Top-Level CPU testbench (tb_RISKV_CPU)
```

> **Note:** Source files and testbenches share the same filenames across the two zip archives. The Sources zip (`TasksNames_2_.zip`) contains the implementation files; the Testbenches zip (`TestNames_1_.zip`) contains the verification files.

---

## Module Breakdown

### Task 1 — ALU (`Abdullah_231000038_Task_1.v`)
**Module:** `ALU` | **Owner:** Abdullah El-Dakhakhny

A 64-bit combinational ALU controlled by a 4-bit `ALUCtrl` signal. Shift operations use the lower 6 bits of operand B as the shift amount. Outputs a `Zero` flag used by branch logic.

| ALUCtrl | Operation |
|---------|-----------|
| `0000`  | AND       |
| `0001`  | OR        |
| `0010`  | ADD       |
| `0110`  | SUB       |
| `1100`  | XOR       |
| `1000`  | SLL       |
| `1001`  | SRL       |
| `1010`  | SRA       |

---

### Task 2 — Control Unit (`Marwan_231000222_Task_2.v`)
**Module:** `C_Unit` | **Owner:** Marwan Negm

Decodes the 32-bit instruction and generates all datapath control signals. The ALU control logic is merged into this module — it resolves the final `AluCtrl` value directly from `opcode`, `funct3`, and `funct7[30]`.

**Outputs:** `branch`, `MemRead`, `MemToReg`, `MemWrite`, `AluSrc`, `RegWrite`, `AluCtrl[3:0]`

| Opcode      | Type     | Instructions                            |
|-------------|----------|-----------------------------------------|
| `0110011`   | R-type   | ADD, SUB, AND, OR, XOR, SLL, SRL, SRA  |
| `0010011`   | I-type   | ADDI, ANDI, ORI, XORI, SLLI, SRLI, SRAI|
| `0000011`   | Load     | LD                                      |
| `0100011`   | Store    | SD                                      |
| `1100011`   | Branch   | BEQ                                     |

---

### Task 3a — Register File (`Ahmed Afifi_231000402_Task_3a.v`)
**Module:** `RegisterFile` | **Owner:** Ahmed Afifi

32 × 64-bit general-purpose registers. `x0` is hardwired to zero via a combinational read guard — writes to register 0 are silently ignored. Reads are asynchronous; writes are synchronous on the rising clock edge, gated by `RegWrite`.

---

### Task 3b — Immediate Generator (`Ahmed Afifi_231000402_Task_3b.v`)
**Module:** `ImmGen` | **Owner:** Ahmed Afifi

Extracts and sign-extends immediates from the 32-bit instruction word based on opcode:

- **I-type / Load** (`0010011`, `0000011`): bits `[31:20]`, sign-extended to 64 bits
- **S-type** (`0100011`): bits `[31:25]` concatenated with `[11:7]`, sign-extended
- **B-type** (`1100011`): bits `[31]`, `[7]`, `[30:25]`, `[11:8]`, with LSB forced to 0

---

### Task 4 — Program Counter (`Steven_231001331_Task_4.v`)
**Module:** `ProgramCounter` | **Owner:** Steven Wilson

A 64-bit register that updates on the positive clock edge. Supports an asynchronous active-high reset that drives `PC_Out` to zero. The next-PC mux (`PC+4` vs. branch target) is implemented externally in the top-level module.

---

### Task 5a — Instruction Memory (`Mark_231000774_Task_5a.v`)
**Module:** `InstructionMemory` | **Owner:** Mark Hany

A 64-entry × 32-bit ROM initialized from `instructions.mem` using `$readmemh`. Word-aligned access: the PC byte address is right-shifted by 2 (`addr >> 2`) to index into the memory array.

> **Setup required:** Update the hardcoded `$readmemh` path to match your local project directory before simulating.

---

### Task 5b — Data Memory (`Mark_231000774_Task_5b.v`)
**Module:** `DataMemory` | **Owner:** Mark Hany

A 64-entry × 64-bit RAM. Reads are asynchronous (combinational, gated by `MemRead`); writes are synchronous on the rising clock edge, gated by `MemWrite`. Addressing uses bits `[8:3]` of the ALU result, enforcing 8-byte (64-bit) word alignment.

---

### Task 5c — Top-Level CPU (`Mark_231000774_Task_5c.v`)
**Module:** `RISKV_CPU` | **Owner:** Mark Hany

Integrates all modules into the complete single-cycle datapath. Key datapath assignments:

```verilog
PC_Plus_4      = PC_Out + 4
PC_Branch      = PC_Out + Imm_Out
Next_PC        = (Branch & Zero) ? PC_Branch : PC_Plus_4
ALU_Src_B      = ALUSrc ? Imm_Out : ReadData2
WriteBack_Data = MemToReg ? Mem_ReadData : ALU_Result
```

Instruction fields are wired directly from `Instruction[31:0]`:

| Field | Bits      | Connected to      |
|-------|-----------|-------------------|
| rs1   | `[19:15]` | `ReadRegister1`   |
| rs2   | `[24:20]` | `ReadRegister2`   |
| rd    | `[11:7]`  | `WriteRegister`   |

---

## Test Program (`instructions.mem`)

The 14-instruction hex program used for integration testing:

```
00700013   // addi  x0,  x0,  7         (no-op: x0 stays 0)
06200813   // addi  x16, x0,  98
00400b13   // addi  x22, x0,  4
016805b3   // add   x11, x16, x22
0105e4b3   // or    x9,  x11, x16
016498b3   // and   x17, x9,  x22
411882b3   // sub   x5,  x17, x17       (result = 0)
01103023   // sd    x17, 0(x0)          (store to mem[0])
00003783   // ld    x15, 0(x0)          (load from mem[0])
01178663   // beq   x15, x17, +12       (taken: x15 == x17)
00788213   // addi  x4,  x17, 7
00000463   // beq   x0,  x0,  +8        (unconditional jump)
00100213   // addi  x4,  x0,  1
00000013   // nop
```

### Expected state after execution

| Register / Memory | Expected Value (decimal) |
|-------------------|--------------------------|
| `x0`              | 0                        |
| `x17`             | 1632                     |
| `x5`              | 0                        |
| `x15`             | 1632                     |
| `x4`              | 1                        |
| `mem[0]`          | 1632                     |

---

## How to Simulate (Vivado)

1. Create a new Vivado project and add all `.v` source files from the Sources archive.
2. Update the `$readmemh` path in `Mark_231000774_Task_5a.v` to point to `instructions.mem` on your machine.
3. Add the desired testbench from the Testbenches archive and set it as the simulation top module.
4. Run behavioral simulation and inspect waveforms or `$display` / `$monitor` console output.

To run the full integration test, use `Mark_231000774_Task_5.v` as the top. It runs 500 ns (50 clock cycles at 100 MHz) and monitors `PC_Out` and `Instruction` throughout execution.

---

## Testbench Summary

| Testbench                               | Module under test | Verification method                                           |
|-----------------------------------------|-------------------|---------------------------------------------------------------|
| `Abdullah_231000038_Task_1.v`           | `ALU`             | `$display` with known operands for all 8 operations; zero flag checked |
| `Marwan_231000222_Task_2.v`             | `C_Unit`          | `$display` of all control signals for all 19 supported instructions |
| `Ahmed Afifi_231000402_Task_3a.v`       | `RegisterFile`    | Write/read x1; attempt write to x0 and confirm it stays zero |
| `Ahmed Afifi_231000402_Task_3b.v`       | `ImmGen`          | Feed `addi`, `ld`, `sd`, `beq` encodings; assert extracted immediates |
| `Steven_231001331_Task_4.v`             | `ProgramCounter`  | `$monitor` through reset, sequential +4, branch-not-taken, branch-taken |
| `Mark_231000774_Task_5.v`               | `RISKV_CPU`       | Run full test program; monitor PC and instruction for 50 cycles |

---

## Future Work

- 5-stage pipeline with hazard detection and data forwarding
- Additional instructions: `JAL`, `JALR`, `LUI`, `AUIPC`
- Extended branch support: `BNE`, `BLT`, `BGE`
- Cache layer for instruction and data memory
