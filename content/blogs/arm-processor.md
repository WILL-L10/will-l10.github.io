---
title: "ARM Processor Implementation"
date: 2024-10-15
draft: false
author: "William Lazcano"
tags:
  - ARM Architecture
  - FPGA
  - Verilog
  - Digital Design
image: /images/projects/arm.jpg
description: "Single-cycle ARM processor with extended instruction set — added EOR and LDRB to a standard ARM core and deployed on FPGA."
toc: true
---

## Overview

This project involved implementing a full single-cycle ARM processor in Verilog and extending it with two new instructions: **EOR** (Exclusive OR) and **LDRB** (Load Register Byte). The processor was deployed on an Altera DE2-115 FPGA and verified end-to-end with a custom ModelSim testbench.

Extending an existing ISA is a different challenge than designing one from scratch — you have to understand exactly how the decoder, ALU, and datapath interact before you can safely add to them without breaking anything.

### Specs at a Glance

| Feature | Detail |
|---|---|
| Architecture | 32-bit single-cycle ARM |
| Custom Instructions | EOR, LDRB |
| Memory Model | Harvard (separate instruction/data) |
| Register File | 16 × 32-bit (R0–R15) |
| Platform | Altera DE2-115 FPGA |
| Clock Speed | 50 MHz |
| Language | Verilog HDL |
| Verification | ModelSim (100% pass rate) |

---

## Adding EOR (Exclusive OR)

EOR performs a bitwise XOR between two registers. The standard ARM ALU uses 2-bit control — adding EOR required extending it to 3 bits and updating the decoder truth table.

```verilog
module alu(
    input  [31:0] a, b,
    input  [2:0]  alu_control,
    output [31:0] result,
    output [3:0]  flags
);
    reg [31:0] alu_out;

    always @(*) begin
        case(alu_control)
            3'b000: alu_out = a + b;   // ADD
            3'b001: alu_out = a - b;   // SUB
            3'b010: alu_out = a & b;   // AND
            3'b011: alu_out = a | b;   // ORR
            3'b100: alu_out = a ^ b;   // EOR (new)
            default: alu_out = 32'b0;
        endcase
    end

    assign result = alu_out;
    assign flags  = {alu_out[31], (alu_out == 32'b0), carry_out, overflow};
endmodule
```

**Decoder update:**

| Funct[4:0] | Instruction | ALU Control |
|---|---|---|
| 00000 | AND | 010 |
| 00001 | **EOR** | **100** |
| 01100 | ORR | 011 |
| 00100 | ADD | 000 |
| 00010 | SUB | 001 |

---

## Adding LDRB (Load Register Byte)

LDRB loads a single byte from memory into a register, zero-extending it to 32 bits. This required adding a byte selector module and a MUX in the memory read path.

```verilog
module byte_selector(
    input  [31:0] word_data,
    input  [1:0]  byte_addr,
    output [31:0] byte_data
);
    always @(*) begin
        case(byte_addr)
            2'b00: byte_data = {24'b0, word_data[7:0]};
            2'b01: byte_data = {24'b0, word_data[15:8]};
            2'b10: byte_data = {24'b0, word_data[23:16]};
            2'b11: byte_data = {24'b0, word_data[31:24]};
        endcase
    end
endmodule
```

```verilog
// MUX: LDR vs LDRB
assign read_data = is_ldrb ? read_data_byte : read_data_word;
```

**Decoder update:**

| Opcode | Funct | Instruction | MemtoReg | ByteLoad |
|---|---|---|---|---|
| 01 | 1 | LDR | 1 | 0 |
| 01 | 1 | **LDRB** | **1** | **1** |
| 01 | 0 | STR | X | 0 |

---

## Top-Level Processor

```verilog
module arm_processor(
    input         clk, reset,
    output [31:0] pc, instr, alu_result
);
    wire [31:0] pc_next, pc_plus4;
    wire [31:0] reg_data1, reg_data2, src_b, result, read_data;
    wire [3:0]  alu_flags;
    wire        reg_write, mem_write, mem_to_reg, alu_src, pc_src;
    wire [2:0]  alu_control;
    wire [1:0]  reg_src;

    register #(32) pc_reg(.clk(clk), .reset(reset), .enable(1'b1), .d(pc_next), .q(pc));
    assign pc_plus4 = pc + 4;
    assign pc_next  = pc_src ? result : pc_plus4;

    instruction_memory imem(.addr(pc), .instr(instr));

    register_file rf(
        .clk(clk), .we3(reg_write),
        .ra1(instr[19:16]), .ra2(instr[3:0]), .wa3(instr[15:12]),
        .wd3(result), .rd1(reg_data1), .rd2(reg_data2)
    );

    assign src_b = alu_src ? {{24{1'b0}}, instr[7:0]} : reg_data2;

    alu alu_unit(.a(reg_data1), .b(src_b), .alu_control(alu_control),
                 .result(alu_result), .flags(alu_flags));

    data_memory dmem(.clk(clk), .we(mem_write), .addr(alu_result),
                     .write_data(reg_data2), .read_data(read_data));

    assign result = mem_to_reg ? read_data : alu_result;

    control_unit cu(
        .opcode(instr[27:25]), .funct(instr[25:20]),
        .rd(instr[15:12]), .flags(alu_flags),
        .reg_write(reg_write), .mem_write(mem_write),
        .mem_to_reg(mem_to_reg), .alu_src(alu_src),
        .alu_control(alu_control), .pc_src(pc_src), .reg_src(reg_src)
    );
endmodule
```

---

## Verification

```assembly
MOV  R1, #0xAA
MOV  R2, #0x55
EOR  R3, R1, R2       ; R3 = 0xAA ^ 0x55 = 0xFF
STR  R3, [R0, #100]
LDRB R4, [R0, #100]   ; R4 = 0x000000FF
LDRB R5, [R0, #101]   ; R5 = 0x00000000
CMP  R3, #0xFF
BEQ  pass
```

**ModelSim Results:**
```
✓ EOR instruction passed
✓ LDRB instruction passed
✓ Flags updated correctly
✓ Memory operations verified
```

**EOR Waveform Check:**

| Signal | Value | Notes |
|---|---|---|
| instr | E0213002 | EOR R3, R1, R2 |
| reg_data1 | 000000AA | R1 |
| reg_data2 | 00000055 | R2 |
| alu_control | 100 | EOR |
| alu_result | 000000FF | Correct |

---

## Timing & Resource Utilization

```
Critical Path:
  Register File Read    2.5 ns
  ALU Computation       8.2 ns
  Memory Access         5.1 ns
  Register Write Setup  2.1 ns
  Total                17.9 ns → Max freq: 55.8 MHz ✓

Resource Usage (Cyclone IV E):
  Logic Elements       3,487 / 114,480   (3.0%)
  Dedicated Registers  1,152 / 114,480
  Memory Bits         16,384 / 3,981,312
```

---

## What I Learned

Extending an ISA taught me to think carefully about side effects. Expanding the ALU control width from 2 to 3 bits meant touching the decoder, the ALU, and verifying that every existing instruction still encoded correctly. One wrong bit in the truth table breaks everything. That kind of careful, systematic thinking carries directly into debugging real hardware.

---

**Code:** [GitHub](https://github.com/Will-L10)
