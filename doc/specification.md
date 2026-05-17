# Design Specification: RISC-V RV32I Pipelined Processor

## 1. Overview

This document specifies the functional behavior of a 32-bit RISC-V processor
implementing the RV32I base integer instruction set (Version 2.1), organized as
a 5-stage in-order pipeline.

## 2. Top-Level Interface

| Port Name        | Direction | Width   | Description                        |
|------------------|-----------|---------|------------------------------------|
| `i_clk`          | Input     | 1       | System clock                       |
| `i_rst_n`        | Input     | 1       | Active-low synchronous reset       |
| `i_instr_data`   | Input     | 32      | Instruction read data from memory  |
| `o_instr_addr`   | Output    | 32      | Instruction fetch address (PC)     |
| `i_data_rdata`   | Input     | 32      | Data memory read data              |
| `o_data_addr`    | Output    | 32      | Data memory address                |
| `o_data_wdata`   | Output    | 32      | Data memory write data             |
| `o_data_we`      | Output    | 1       | Data memory write enable           |
| `o_data_be`      | Output    | 4       | Data memory byte enable            |

## 3. Instruction Set Summary

The processor implements all 37 instructions in the RV32I base integer ISA.

### 3.1 R-Type (Register-Register)

| Instruction | Opcode    | funct3 | funct7    | Operation               |
|-------------|-----------|--------|-----------|-------------------------|
| ADD         | 0110011   | 000    | 0000000   | rd = rs1 + rs2          |
| SUB         | 0110011   | 000    | 0100000   | rd = rs1 - rs2          |
| AND         | 0110011   | 111    | 0000000   | rd = rs1 & rs2          |
| OR          | 0110011   | 110    | 0000000   | rd = rs1 \| rs2         |
| XOR         | 0110011   | 100    | 0000000   | rd = rs1 ^ rs2          |
| SLL         | 0110011   | 001    | 0000000   | rd = rs1 << rs2[4:0]    |
| SRL         | 0110011   | 101    | 0000000   | rd = rs1 >> rs2[4:0]    |
| SRA         | 0110011   | 101    | 0100000   | rd = rs1 >>> rs2[4:0]   |
| SLT         | 0110011   | 010    | 0000000   | rd = (rs1 < rs2) ? 1:0 (signed)  |
| SLTU        | 0110011   | 011    | 0000000   | rd = (rs1 < rs2) ? 1:0 (unsigned)|

### 3.2 I-Type (Immediate)

| Instruction | Opcode    | funct3 | Operation                   |
|-------------|-----------|--------|-----------------------------|
| ADDI        | 0010011   | 000    | rd = rs1 + imm[11:0]        |
| ANDI        | 0010011   | 111    | rd = rs1 & imm[11:0]        |
| ORI         | 0010011   | 110    | rd = rs1 \| imm[11:0]       |
| XORI        | 0010011   | 100    | rd = rs1 ^ imm[11:0]        |
| SLTI        | 0010011   | 010    | rd = (rs1 < imm) ? 1:0 (signed)  |
| SLTIU       | 0010011   | 011    | rd = (rs1 < imm) ? 1:0 (unsigned)|
| SLLI        | 0010011   | 001    | rd = rs1 << imm[4:0]        |
| SRLI        | 0010011   | 101    | rd = rs1 >> imm[4:0]        |
| SRAI        | 0010011   | 101    | rd = rs1 >>> imm[4:0]       |
| LW          | 0000011   | 010    | rd = Mem[rs1 + imm]         |
| LH          | 0000011   | 001    | rd = SignExt(Mem[rs1+imm][15:0]) |
| LHU         | 0000011   | 101    | rd = ZeroExt(Mem[rs1+imm][15:0]) |
| LB          | 0000011   | 000    | rd = SignExt(Mem[rs1+imm][7:0])  |
| LBU         | 0000011   | 100    | rd = ZeroExt(Mem[rs1+imm][7:0])  |
| JALR        | 1100111   | 000    | rd = PC+4; PC = (rs1+imm) & ~1   |

### 3.3 S-Type (Store)

| Instruction | Opcode    | funct3 | Operation                   |
|-------------|-----------|--------|-----------------------------|
| SW          | 0100011   | 010    | Mem[rs1 + imm] = rs2        |
| SH          | 0100011   | 001    | Mem[rs1 + imm][15:0] = rs2[15:0] |
| SB          | 0100011   | 000    | Mem[rs1 + imm][7:0] = rs2[7:0]   |

### 3.4 B-Type (Branch)

| Instruction | Opcode    | funct3 | Operation                        |
|-------------|-----------|--------|----------------------------------|
| BEQ         | 1100011   | 000    | if (rs1 == rs2) PC += imm        |
| BNE         | 1100011   | 001    | if (rs1 != rs2) PC += imm        |
| BLT         | 1100011   | 100    | if (rs1 < rs2) PC += imm (signed)|
| BGE         | 1100011   | 101    | if (rs1 >= rs2) PC += imm (signed)|
| BLTU        | 1100011   | 110    | if (rs1 < rs2) PC += imm (unsigned)|
| BGEU        | 1100011   | 111    | if (rs1 >= rs2) PC += imm (unsigned)|

### 3.5 U-Type (Upper Immediate)

| Instruction | Opcode    | Operation                        |
|-------------|-----------|----------------------------------|
| LUI         | 0110111   | rd = imm[31:12] << 12            |
| AUIPC       | 0010111   | rd = PC + (imm[31:12] << 12)     |

### 3.6 J-Type (Jump)

| Instruction | Opcode    | Operation                        |
|-------------|-----------|----------------------------------|
| JAL         | 1101111   | rd = PC+4; PC += imm             |

## 4. Pipeline Architecture

### 4.1 Stage Descriptions

**IF (Instruction Fetch):**
- Drives PC to instruction memory
- Fetches 32-bit instruction
- Increments PC by 4 (default next address)
- Accepts branch/jump target from EX stage

**ID (Instruction Decode):**
- Decodes instruction opcode, funct3, funct7
- Reads two source registers from register file
- Generates immediate value (sign-extended)
- Generates control signals for downstream stages

**EX (Execute):**
- Performs ALU operation
- Computes branch target address and condition
- Selects forwarded operands when applicable

**MEM (Memory Access):**
- Performs load/store operations via data memory interface
- Passes ALU result through for non-memory instructions

**WB (Write Back):**
- Writes result to destination register
- Selects between ALU result and memory read data

### 4.2 Pipeline Registers

| Register    | Fields                                                     |
|-------------|-------------------------------------------------------------|
| IF/ID       | instruction, PC, PC+4                                       |
| ID/EX       | rs1_data, rs2_data, immediate, rd_addr, control signals, PC |
| EX/MEM      | alu_result, rs2_data (for stores), rd_addr, control signals |
| MEM/WB      | alu_result, mem_read_data, rd_addr, control signals          |

### 4.3 Hazard Handling

**Data hazards:** Resolved via a forwarding unit (EX-to-EX, MEM-to-EX). Load-use
hazard requires a one-cycle stall inserted by the hazard detection unit.

**Control hazards:** Static branch-not-taken prediction. On a taken branch, flush
the IF/ID and ID/EX pipeline registers (two-cycle penalty).

**Structural hazards:** None by design - separate instruction and data memory ports.

## 5. Register File

- 32 registers, x0 hardwired to zero
- Dual read ports (rs1, rs2), single write port (rd)
- Write-first behavior: if reading and writing the same register in the same
  cycle, the new value is forwarded

## 6. Reset Behavior

- Active-low synchronous reset (`i_rst_n`)
- On reset: PC = 0x00000000, all pipeline registers cleared, register file zeroed

## 7. Design Constraints

- Target clock frequency: 100 MHz
- Target FPGA: Xilinx Artix-7 (XC7A35T)
- No multicycle or pipelined multiplier/divider (RV32I does not include M extension)
