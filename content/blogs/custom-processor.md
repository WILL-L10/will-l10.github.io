---
title: "Custom 8-Bit Multicycle Processor"
date: 2024-11-16
draft: false
author: "William Lazcano"
tags:
  - SystemVerilog
  - FPGA
  - Processor Design
  - Digital Systems
  - FSM
image: /images/projects/processor.jpg
description: "Custom 8-bit processor from scratch with 16-instruction ISA and FSM-based control unit"
toc: true
---

## Project Overview

Designed and implemented a complete custom processor from scratch with 16-instruction ISA and FSM-based control unit, capable of executing complex matrix operations entirely in software.

## Architecture

### Instruction Set Architecture (ISA)
Custom 16-instruction set including:
- **Arithmetic**: ADD, SUB, MULT, DIV
- **Logic**: AND, OR, XOR, NOT
- **Control Flow**: JUMP, conditional branches (BEQ, BNE, BLT, BGT)
- **Memory**: LOAD, STORE
- **Special**: NOP, HALT

### Control Unit
- Finite State Machine (FSM) based control
- Multi-cycle execution with 5-7 cycles per instruction
- State-based instruction decode and execution
- Comprehensive flag management (Zero, Negative, Carry, Overflow)

## Implementation

### Hardware Platform
- **FPGA**: Altera DE2-115
- **Memory**: 512-byte unified instruction/data memory
- **Display**: Real-time 7-segment display showing PC, opcode, accumulator
- **Programming**: Manual mode using FPGA switches for direct memory manipulation

### Key Features
1. **Matrix Operations**: Successfully executed 2×2 matrix operations
   - Addition
   - Subtraction
   - Multiplication
   - Inversion (using integer arithmetic)

2. **Debugging Features**
   - Pause-on-write capability
   - Real-time register display
   - Step-by-step execution mode
   - Memory inspection via switches

3. **Programming Modes**
   - Automatic execution from memory
   - Manual programming using switches
   - Single-step debug mode

## Code Example
```systemverilog
module smm_processor (
    input clk,
    input reset,
    input [7:0] manual_data,
    input manual_write,
    output [7:0] pc_out,
    output [7:0] accumulator_out,
    output [3:0] state_out
);
    // State machine states
    typedef enum logic [3:0] {
        FETCH = 4'b0000,
        DECODE = 4'b0001,
        EXECUTE = 4'b0010,
        MEMORY = 4'b0011,
        WRITEBACK = 4'b0100,
        HALT_STATE = 4'b0101
    } state_t;
    
    state_t current_state, next_state;
    
    // Registers
    logic [7:0] pc;
    logic [7:0] ir;
    logic [7:0] accumulator;
    logic [7:0] temp_reg;
    
    // ALU
    logic [7:0] alu_result;
    logic zero_flag, negative_flag;
    
    // Control signals
    logic pc_write, acc_write, mem_write;
    logic [3:0] alu_op;
    
    // State register
    always_ff @(posedge clk or posedge reset) begin
        if (reset)
            current_state <= FETCH;
        else
            current_state <= next_state;
    end
    
    // FSM logic
    always_comb begin
        case (current_state)
            FETCH: begin
                pc_write = 1;
                next_state = DECODE;
            end
            DECODE: begin
                next_state = EXECUTE;
            end
            EXECUTE: begin
                case (ir[7:4])  // Opcode
                    4'b0000: alu_op = 4'b0000;  // ADD
                    4'b0001: alu_op = 4'b0001;  // SUB
                    4'b0010: alu_op = 4'b0010;  // MULT
                    4'b0011: alu_op = 4'b0011;  // DIV
                    default: alu_op = 4'b0000;
                endcase
                next_state = WRITEBACK;
            end
            WRITEBACK: begin
                acc_write = 1;
                next_state = FETCH;
            end
            HALT_STATE: begin
                next_state = HALT_STATE;
            end
        endcase
    end
endmodule
```

## Testing & Results

### Test Programs
1. **Matrix Addition**: Successfully computed sum of two 2×2 matrices
2. **Matrix Multiplication**: Verified correct results for 2×2 matrix product
3. **Matrix Inversion**: Implemented using integer arithmetic operations

### Performance Metrics
- **Clock Speed**: 50 MHz
- **Instruction Throughput**: ~10 million instructions/second
- **Memory Access**: Single-cycle read/write
- **Average CPI**: 5.5 cycles per instruction

### Verification
- Comprehensive testbenches in ModelSim
- Waveform analysis for all instructions
- Physical testing on FPGA with manual verification
- 100% functional correctness achieved

## Challenges & Solutions

### Challenge 1: Integer Matrix Inversion
**Problem**: Floating-point operations not available
**Solution**: Implemented fixed-point arithmetic using scaled integers and careful attention to overflow

### Challenge 2: Memory Management
**Problem**: Limited 512-byte memory space
**Solution**: Optimized instruction encoding and data layout, used accumulator architecture to minimize register file

### Challenge 3: Debugging
**Problem**: Difficult to observe internal state
**Solution**: Implemented comprehensive 7-segment display output and pause-on-write functionality

## Skills Demonstrated
- Digital system architecture
- FSM design and implementation
- SystemVerilog HDL programming
- FPGA synthesis and deployment
- Algorithm optimization for hardware
- Hardware debugging techniques
- Instruction set architecture design

## Future Enhancements
- Pipeline implementation for higher throughput
- Cache memory system
- Interrupt handling
- Extended instruction set with floating-point support
- Peripheral interface (UART, GPIO)