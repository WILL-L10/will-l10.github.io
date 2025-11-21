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
image: /images/projects/arm-processor.jpg
description: "Extended ARM processor with additional instructions implemented on FPGA"
toc: true
---

## Project Overview

Implemented an ARM processor with extended instruction set on FPGA hardware, demonstrating proficiency in computer architecture and hardware description languages.

## Technical Details

### Architecture
- **Instruction Set**: Extended ARM ISA with custom instructions
- **Pipeline**: Multi-stage pipeline design
- **Memory**: Harvard architecture with separate instruction and data memory

### Implementation
- Developed using Verilog HDL
- Synthesized and tested on Altera DE2-115 FPGA board
- Clock frequency: 50 MHz
- Verified functionality through comprehensive testbenches

### Key Features
- Implemented standard ARM instructions (ADD, SUB, AND, ORR, etc.)
- Added custom instructions for specialized operations
- Integrated memory-mapped I/O
- Hardware debugging using SignalTap II

## Code Example
```verilog
module arm_processor (
    input clk,
    input reset,
    output [31:0] pc,
    output [31:0] instr
);
    // Datapath components
    wire [31:0] alu_result;
    wire [3:0] alu_flags;
    
    // Control signals
    wire reg_write;
    wire mem_write;
    wire [1:0] alu_op;
    
    // Instantiate datapath and control unit
    datapath dp(
        .clk(clk),
        .reset(reset),
        .reg_write(reg_write),
        .alu_result(alu_result)
    );
    
    control_unit cu(
        .opcode(instr[27:25]),
        .reg_write(reg_write),
        .mem_write(mem_write)
    );
endmodule
```

## Testing & Verification

Comprehensive testing methodology:
- Unit tests for individual components (ALU, Register File, Control Unit)
- Integration testing with complete instruction sequences
- Validation against ARM specification
- Performance benchmarking

## Results

Successfully implemented and tested all required instructions with 100% functional correctness on FPGA hardware.

## Skills Demonstrated
- Computer architecture design
- Verilog HDL programming
- FPGA synthesis and implementation
- Digital system verification
- Hardware debugging