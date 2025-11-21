---
title: "MIPS Processor with Extended ISA"
date: 2024-09-20
draft: false
author: "William Lazcano"
tags:
  - MIPS
  - FPGA
  - SystemVerilog
  - Pipelining
image: /images/projects/mips-processor.jpg
description: "5-stage pipelined MIPS processor with hazard detection and forwarding"
toc: true
---

## Project Overview

Designed and implemented a 5-stage pipelined MIPS processor with hazard detection, data forwarding, and an extended instruction set on FPGA.

## Architecture

### Pipeline Stages
1. **Instruction Fetch (IF)**: Retrieves instruction from memory
2. **Instruction Decode (ID)**: Decodes instruction and reads registers
3. **Execute (EX)**: Performs ALU operations
4. **Memory (MEM)**: Accesses data memory
5. **Write Back (WB)**: Writes results to register file

### Hazard Handling
- **Data Hazards**: Implemented forwarding paths to eliminate stalls
- **Control Hazards**: Branch prediction with flush mechanism
- **Structural Hazards**: Resolved through separate instruction/data memory

## Implementation Details

### Extended Instructions
Added custom instructions beyond standard MIPS ISA:
- `MULT`: 32-bit multiplication
- `DIV`: Integer division
- `SWAP`: Register swap operation
- Custom branch conditions

### Hardware Specifications
- **Platform**: Altera DE2-115 FPGA
- **Language**: SystemVerilog
- **Clock Speed**: 50 MHz
- **Register File**: 32 x 32-bit registers

## Code Snippet
```systemverilog
module mips_pipeline (
    input logic clk,
    input logic reset,
    output logic [31:0] pc_out,
    output logic [31:0] instr_out
);
    // Pipeline registers
    logic [31:0] if_id_instr, if_id_pc;
    logic [31:0] id_ex_rs, id_ex_rt;
    logic [31:0] ex_mem_alu_result;
    logic [31:0] mem_wb_read_data;
    
    // Hazard detection unit
    hazard_unit hu (
        .rs_id(if_id_instr[25:21]),
        .rt_id(if_id_instr[20:16]),
        .rt_ex(id_ex_rt),
        .stall(stall),
        .flush(flush)
    );
    
    // Forwarding unit
    forwarding_unit fu (
        .rs_ex(id_ex_rs_addr),
        .rt_ex(id_ex_rt_addr),
        .rd_mem(ex_mem_rd),
        .rd_wb(mem_wb_rd),
        .forward_a(forward_a),
        .forward_b(forward_b)
    );
endmodule
```

## Performance Analysis

### Metrics
- **CPI (Cycles Per Instruction)**: ~1.2 (with forwarding)
- **Pipeline Efficiency**: 83%
- **Branch Prediction Accuracy**: 78%

### Benchmarks
Tested with various programs:
- Bubble sort algorithm
- Matrix multiplication
- Fibonacci sequence calculation

## Testing Methodology

1. **Modular Testing**: Each pipeline stage tested independently
2. **Integration Testing**: Full pipeline validation
3. **Hazard Testing**: Specific test cases for data/control hazards
4. **Performance Testing**: Benchmark programs with timing analysis

## Skills Demonstrated
- Advanced digital design
- Pipelining and hazard resolution
- SystemVerilog programming
- FPGA implementation
- Performance optimization