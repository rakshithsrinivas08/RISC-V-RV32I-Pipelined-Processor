# Architecture Document: RISC-V RV32I Pipelined Processor

## 1. Module Hierarchy
![Block Diagram](doc/top_level_block_diagram.svg)
## 2. Module Descriptions

### 2.1 `rv32i_top`
Top-level module. Instantiates all pipeline stages, pipeline registers,
forwarding unit, and hazard detection unit. Exposes instruction and data
memory interfaces to the external system.

### 2.2 `program_counter`
32-bit register holding the current PC. On each clock cycle, loads either
PC+4 (sequential), branch target, or jump target based on control input.
Resets to 0x00000000. Stalls when the hazard detection unit asserts `stall`.

### 2.3 `control_unit`
Combinational logic. Decodes the 7-bit opcode (and funct3/funct7 for ALU
operations) to produce control signals: `alu_op`, `alu_src`, `mem_read`,
`mem_write`, `reg_write`, `mem_to_reg`, `branch`, `jump`.

### 2.4 `register_file`
32x32-bit register file. Port A reads rs1, Port B reads rs2, Port W writes
rd. Register x0 always returns zero. Write-first forwarding is implemented
so that a read and write to the same register in the same cycle returns the
new value.

### 2.5 `immediate_generator`
Extracts and sign-extends the immediate value from the instruction based on
instruction type (I, S, B, U, J). Purely combinational.

### 2.6 `alu`
Performs arithmetic and logic operations based on a 4-bit `alu_control` signal.
Outputs the 32-bit result and a zero flag.

| alu_control | Operation |
|-------------|-----------|
| 0000        | ADD       |
| 0001        | SUB       |
| 0010        | AND       |
| 0011        | OR        |
| 0100        | XOR       |
| 0101        | SLL       |
| 0110        | SRL       |
| 0111        | SRA       |
| 1000        | SLT       |
| 1001        | SLTU      |

### 2.7 `branch_unit`
Evaluates branch conditions (BEQ, BNE, BLT, BGE, BLTU, BGEU) by comparing
the two source operands. Outputs a `branch_taken` signal.

### 2.8 `forwarding_unit`
Detects data dependencies between the EX stage and the two preceding
instructions (in MEM and WB stages). Drives mux selects for ALU input A
and input B to choose between register file output, EX/MEM forwarded value,
or MEM/WB forwarded value.

### 2.9 `hazard_detection_unit`
Detects load-use hazards (instruction in EX stage is a load, and the
instruction in ID stage reads the load's destination register). When
detected, stalls the IF and ID stages for one cycle and inserts a NOP
(bubble) into the EX stage.

## 3. Pipeline Diagram

_(To be replaced with PNG from draw.io)_

## 4. Signal Naming Convention

| Prefix | Meaning                          | Example            |
|--------|----------------------------------|--------------------|
| `i_`   | Input port                       | `i_clk`            |
| `o_`   | Output port                      | `o_alu_result`     |
| `w_`   | Combinational wire               | `w_branch_target`  |
| `r_`   | Registered signal (flip-flop)    | `r_pc`             |
| `c_`   | Control signal                   | `c_reg_write`      |

Pipeline stage prefixes in internal signals:

| Prefix   | Stage                |
|----------|----------------------|
| `if_`    | Instruction Fetch    |
| `id_`    | Instruction Decode   |
| `ex_`    | Execute              |
| `mem_`   | Memory Access        |
| `wb_`    | Write Back           |

## 5. Clock and Reset Strategy

- Single clock domain (`i_clk`), positive-edge triggered
- Synchronous active-low reset (`i_rst_n`)
- All pipeline registers and the PC reset synchronously

## 6. Design Decisions Log

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Reset type | Synchronous active-low | Industry standard for ASIC; avoids async reset routing issues |
| Branch resolution | EX stage | Simpler than ID-stage resolution; 2-cycle penalty is acceptable for RV32I |
| Branch prediction | Static not-taken | Simplest to implement; upgrade path to BTB exists |
| Memory interface | Split I-mem / D-mem | Eliminates structural hazards; Harvard architecture |
| Register file write | Write-first | Avoids needing a forwarding path from WB to ID |
