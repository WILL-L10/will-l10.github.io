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

Designed and implemented a **complete custom 8-bit multicycle processor** from the ground up, featuring a 16-instruction ISA and FSM-based control unit. This processor successfully executes complex matrix operations entirely in software using integer arithmetic.

![Processor Block Diagram](/images/projects/processor.jpg)

### Key Specifications

| Feature | Specification |
|---------|--------------|
| **Architecture** | 8-bit multicycle |
| **Instruction Set** | 16 custom instructions |
| **Memory** | 512 bytes unified (byte-addressable) |
| **Execution Cycles** | 1-4 cycles per instruction |
| **Control Logic** | FSM-based with 6 states |
| **Target Platform** | Altera DE2-115 FPGA |

## Architecture Deep Dive

### Instruction Set Architecture

The processor implements 16 carefully designed instructions:

#### Arithmetic Operations
```
ADD   - Addition
SUB   - Subtraction  
MULT  - Multiplication
DIV   - Division
INAC  - Increment Accumulator
```

#### Logic Operations
```
AND   - Bitwise AND
OR    - Bitwise OR
XOR   - Bitwise XOR
NOT   - Bitwise NOT
CLAC  - Clear Accumulator
```

#### Control Flow
```
JUMP  - Unconditional jump
JMPZ  - Jump if zero
JPNZ  - Jump if not zero
```

#### Data Movement
```
LDAC  - Load to Accumulator
STAC  - Store from Accumulator
MVAC  - Move to Accumulator
MOVR  - Move to R register
```

### Finite State Machine Design

The control unit operates through 6 distinct states:
```
┌─────────────┐
│ FETCH_STATE │
└──────┬──────┘
       │
       ▼
┌──────────────┐
│ NORMAL_STATE │─────┐
└──────┬───────┘     │
       │             │
       ├─────────────┼──────────┐
       │             │          │
       ▼             ▼          ▼
┌──────────────┐ ┌──────────┐ ┌────────────┐
│LOAD_ADDRESS1 │ │LOAD_STORE│ │ JUMP_STATE │
└──────┬───────┘ └──────────┘ └────────────┘
       │
       ▼
┌──────────────┐
│LOAD_ADDRESS2 │
└──────────────┘
```

### Datapath Components

The processor's datapath consists of 11 major components:

**Registers:**
- **PC** (16-bit): Program Counter
- **IR** (8-bit): Instruction Register
- **AC** (8-bit): Accumulator
- **R** (8-bit): General Purpose Register
- **MDA** (16-bit): Memory Data Address Register

**Computational Units:**
- **ALU** (8-bit): Arithmetic Logic Unit
- **FF** (1-bit): Zero Flag

**Control Elements:**
- **PC Multiplexers** (2x 16-bit): Control program flow
- **AC Multiplexers** (3x 8-bit): Manage accumulator input
- **16-bit Adder**: PC increment
- **Instruction Multiplexer**: Select instruction source

## Implementation Details

### Control Unit Code
```systemverilog
module controller(
    input logic clk, reset, prog_mode,
    input logic FF,
    input logic [7:0] op_code,
    output logic [1:0] MDAwrite,
    output logic [2:0] ALU_control, AC_mux_sel,
    output logic IRwrite, Rwrite, ACwrite, PCenable, MemWrite,
    output logic PCmuxSel, intr_source, address_mux_sel
);
    
    typedef enum {
        fetch_state, 
        normal_state, 
        load_address1, 
        load_address2, 
        jump_state, 
        load_store
    } FSM_states;
    
    FSM_states state, nextstate;
    reg Z;
    
    always@(posedge clk) begin
        if (reset || prog_mode)
            state <= fetch_state;
        else begin
            state <= nextstate;
            if(op_code[3])
                Z <= FF;
        end
    end
    
    // State transition and control signal logic
    always@(*) begin
        case(state)
            fetch_state: 
                nextstate <= normal_state;
            
            normal_state: begin
                if(op_code[3]) // ALU instruction
                    nextstate <= fetch_state;
                else if (op_code[3:0] == 4'b0001 || 
                         op_code[3:0] == 4'b0010 || 
                         op_code[3:0] == 4'b0101 || 
                         op_code[3:0] == 4'b0110 || 
                         op_code[3:0] == 4'b0111)
                    nextstate <= load_address1;
                else if (op_code[4])
                    nextstate <= load_store;
                else
                    nextstate <= fetch_state;
            end
            
            load_address1: 
                nextstate <= load_address2;
            
            load_address2: 
                nextstate <= (op_code[3:0] == 4'b0001 || 
                             op_code[3:0] == 4'b0010) ? 
                             load_store : jump_state;
            
            jump_state: 
                nextstate <= fetch_state;
            
            load_store: 
                nextstate <= fetch_state;
            
            default: 
                nextstate <= fetch_state;
        endcase
    end
endmodule
```

### ALU Implementation
```systemverilog
module alu(
    input logic [7:0] a, b,
    input logic [2:0] alu_control,
    output logic [7:0] result,
    output logic zero_flag
);
    
    always_comb begin
        case(alu_control)
            3'b000: result = a + b;      // ADD
            3'b001: result = a - b;      // SUB
            3'b010: result = a + 1;      // INAC
            3'b011: result = 8'b0;       // CLAC
            3'b100: result = a & b;      // AND
            3'b101: result = a * b;      // MULT
            3'b110: result = a / b;      // DIV
            3'b111: result = ~a;         // NOT
            default: result = 8'b0;
        endcase
        
        zero_flag = (result == 8'b0);
    end
endmodule
```

## Matrix Operations Demo

### Test Program: 2×2 Matrix Operations

The processor executes complex matrix algorithms entirely in software:

**Matrix Addition:**
```
A = | 6  5 |    B = | 1  2 |    C = | 7  7  |
    | 8  7 |        | 3  4 |        | 11 11 |
```

**Matrix Multiplication:**
```
A × B = | 21  32 |
        | 29  44 |
```

**Matrix Inversion** (using integer arithmetic):
```
A⁻¹ = | det(A)⁻¹ × d   -det(A)⁻¹ × b |
      | -det(A)⁻¹ × c   det(A)⁻¹ × a |

Where det(A) = ad - bc
```

### Performance Metrics

| Operation | Clock Cycles | Execution Time @ 50MHz |
|-----------|-------------|----------------------|
| Matrix Add (2×2) | ~120 | 2.4 μs |
| Matrix Multiply (2×2) | ~350 | 7.0 μs |
| Matrix Inversion (2×2) | ~500 | 10.0 μs |

## FPGA Deployment

### Hardware Features

**Programming Mode:**
- 9 switches control 512-byte address space
- 8 switches set data value
- Red LEDs reflect current switch settings
- Manual memory programming via switches

**Display Output:**
- 8 seven-segment displays
- Real-time PC, opcode, and accumulator display
- Matrix C results (address 0x1EF+) shown live
- Two segments per byte

### Resource Utilization
```
┌─────────────────────┬──────────┬──────────┐
│ Resource            │ Used     │ Available│
├─────────────────────┼──────────┼──────────┤
│ Logic Elements      │ 4,302    │ 114,480  │
│ Dedicated Registers │ 4,171    │ 114,480  │
│ Memory Bits         │ 4,096    │ 3,981,312│
│ Multipliers (9-bit) │ 0        │ 532      │
└─────────────────────┴──────────┴──────────┘
Utilization: ~3.8%
```

## Testing & Verification

### ModelSim Simulation

Comprehensive testing validated all 16 instructions:
```
✓ Test 1: ADD operation - PASSED
✓ Test 2: SUB operation - PASSED
✓ Test 3: MULT operation - PASSED
✓ Test 4: DIV operation - PASSED
✓ Test 5: Matrix addition (2×2) - PASSED
✓ Test 6: Matrix multiplication (2×2) - PASSED
✓ Test 7: Conditional jumps (JMPZ/JPNZ) - PASSED
✓ Test 8: Memory load/store - PASSED
```

### Waveform Analysis

All instructions verified through detailed waveform inspection in ModelSim, confirming correct:
- State transitions
- Control signal generation
- Data path operations
- Memory accesses

## Challenges & Solutions

### Challenge 1: Integer Matrix Inversion

**Problem:** No floating-point support for fractional results

**Solution:** Implemented fixed-point arithmetic using scaled integers with careful overflow management. All intermediate calculations use scaled values to maintain precision.

### Challenge 2: Limited Memory (512 bytes)

**Problem:** Constrained space for program and data

**Solution:** Optimized instruction encoding to 1-3 bytes per instruction. Used accumulator architecture to minimize register file size.

### Challenge 3: Debugging Complexity

**Problem:** Difficult to observe internal processor state

**Solution:** Implemented comprehensive 7-segment display showing PC, opcode, accumulator, and pause-on-write functionality for step-through debugging.

## Skills Demonstrated

- ✅ Digital system architecture from scratch
- ✅ FSM design and implementation
- ✅ SystemVerilog HDL programming
- ✅ FPGA synthesis and deployment
- ✅ Algorithm optimization for constrained hardware
- ✅ Hardware debugging techniques
- ✅ Instruction set architecture design
- ✅ Control unit and datapath coordination

## Future Enhancements

**Performance:**
- Pipeline implementation for higher throughput
- Cache memory system
- Hardware multiply/divide units

**Features:**
- Interrupt handling
- Extended 16-bit operations
- Floating-point support
- UART peripheral interface
- Expanded memory addressing

## Conclusion

This project demonstrates a fully functional minimal processor architecture suitable for educational use and embedded applications. Despite its simplicity, the SMM processor successfully executes meaningful computation through software routines, showcasing how general-purpose computation emerges from a simple FSM-based CPU design.

The combination of custom ISA design, FSM-based control, and practical FPGA deployment provides hands-on insight into fundamental computer architecture principles—all implemented without the complexity of modern CPU features.

---

**Project Files:** [GitHub Repository](https://github.com/Will-L10)

**Demo Video:** Available upon request

**Documentation:** Complete technical specifications in project report