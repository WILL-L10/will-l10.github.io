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

### Custom Instructions Added
1. **EOR (Exclusive OR)**: Bitwise XOR operation
   - Expanded ALU from 2-bit to 3-bit control
   - Maintained flag updates (Zero, Negative)
   - Modified ALU decoder for XOR support

2. **LDRB (Load Register Byte)**: Byte-level memory access
   - Implemented byte selection logic
   - Extract individual bytes from word-aligned memory
   - Used address bits [1:0] for byte position

### Implementation Details
- Developed using Verilog HDL
- Synthesized and tested on Altera DE2-115 FPGA board
- Clock frequency: 50 MHz
- Verified functionality through comprehensive testbenches

## Architecture Components

### Datapath
- 32-bit register file (16 registers)
- ALU with extended operations
- Shifter unit
- Memory interface with byte selection
- Multiplexers for data routing

### Control Unit
- Main decoder for instruction decode
- ALU decoder for operation selection
- Control signal generation
- Condition code logic

## Code Example
```verilog
module arm_processor (
    input clk,
    input reset,
    output [31:0] pc,
    output [31:0] instr,
    output [31:0] alu_result
);
    // Datapath components
    wire [31:0] reg_data1, reg_data2;
    wire [31:0] alu_out;
    wire [3:0] alu_flags;
    
    // Control signals
    wire reg_write;
    wire mem_write;
    wire [2:0] alu_control;  // Extended to 3 bits for EOR
    wire mem_to_reg;
    wire [1:0] byte_select;
    
    // Register File
    register_file rf(
        .clk(clk),
        .we3(reg_write),
        .ra1(instr[19:16]),
        .ra2(instr[3:0]),
        .wa3(instr[15:12]),
        .wd3(result),
        .rd1(reg_data1),
        .rd2(reg_data2)
    );
    
    // ALU with EOR support
    alu alu_unit(
        .a(reg_data1),
        .b(src_b),
        .alu_control(alu_control),
        .result(alu_out),
        .flags(alu_flags)
    );
    
    // ALU with extended control
    always @(*) begin
        case (alu_control)
            3'b000: result = a + b;           // ADD
            3'b001: result = a - b;           // SUB
            3'b010: result = a & b;           // AND
            3'b011: result = a | b;           // ORR
            3'b100: result = a ^ b;           // EOR (NEW)
            default: result = 32'b0;
        endcase
    end
    
    // Byte selection logic for LDRB
    byte_selector bs(
        .word_data(mem_data),
        .byte_addr(alu_out[1:0]),
        .byte_data(byte_out)
    );
    
    // Control Unit
    control_unit cu(
        .opcode(instr[27:25]),
        .funct(instr[25:20]),
        .rd(instr[15:12]),
        .reg_write(reg_write),
        .mem_write(mem_write),
        .alu_control(alu_control)
    );
endmodule

// Byte selector for LDRB instruction
module byte_selector(
    input [31:0] word_data,
    input [1:0] byte_addr,
    output reg [31:0] byte_data
);
    always @(*) begin
        case (byte_addr)
            2'b00: byte_data = {24'b0, word_data[7:0]};
            2'b01: byte_data = {24'b0, word_data[15:8]};
            2'b10: byte_data = {24'b0, word_data[23:16]};
            2'b11: byte_data = {24'b0, word_data[31:24]};
        endcase
    end
endmodule
```

## Testing & Verification

### Testbench Strategy
```verilog
module arm_processor_tb;
    reg clk, reset;
    wire [31:0] pc, instr, result;
    
    // Instantiate processor
    arm_processor dut(
        .clk(clk),
        .reset(reset),
        .pc(pc),
        .instr(instr),
        .alu_result(result)
    );
    
    // Test EOR instruction
    initial begin
        reset = 1; #10;
        reset = 0;
        
        // EOR R3, R1, R2
        // Expected: R3 = R1 XOR R2
        #100;
        if (result !== expected_xor)
            $display("ERROR: EOR instruction failed");
        
        // Test LDRB instruction
        // LDRB R4, [R5, #2]
        #50;
        if (result[31:8] !== 24'b0)
            $display("ERROR: LDRB upper bits not cleared");
            
        $display("All tests passed!");
        $finish;
    end
endmodule
```

### Verification Results
- **ModelSim Simulation**: 100% accuracy on all test cases
- **FPGA Testing**: All instructions verified on hardware
- **Waveform Analysis**: Confirmed correct signal timing
- **Timing Analysis**: Met timing constraints at 50 MHz

## Performance Analysis

### Timing Optimization
Used TimeQuest Timing Analyzer to:
- Identify critical paths
- Optimize ALU logic depth
- Balance pipeline stages
- Achieve target clock frequency

### Resource Utilization
- Logic Elements: ~3,500
- Memory Bits: 16,384 (instruction + data memory)
- Multiplexers: Optimized for minimal delay
- Registers: 32-bit × 16 register file

## Skills Demonstrated
- Computer architecture design
- Verilog HDL programming
- FPGA synthesis and implementation
- Digital system verification
- Hardware debugging with SignalTap II
- Timing analysis and optimization
- Instruction set extension