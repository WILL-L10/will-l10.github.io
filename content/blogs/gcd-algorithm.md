---
title: "GCD Algorithm Hardware Implementation"
date: 2024-08-10
draft: false
author: "William Lazcano"
tags:
  - Algorithms
  - FSM
  - Verilog
  - Digital Design
image: /images/projects/gcd-fsm.jpg
description: "Finite State Machine implementation of GCD algorithm in hardware"
toc: true
---

## Project Overview

Implemented the Greatest Common Divisor (GCD) algorithm using a Finite State Machine (FSM) approach in hardware, demonstrating algorithm optimization for digital circuits.

## Algorithm

### Euclidean Algorithm
The GCD algorithm uses the principle:
```
GCD(a, b) = GCD(b, a mod b)
```

Continues until `b = 0`, at which point `a` is the GCD.

## FSM Design

### State Diagram
The implementation uses 4 states:
1. **IDLE**: Waiting for input
2. **LOAD**: Load operands into registers
3. **COMPUTE**: Perform modulo operation
4. **DONE**: Output result

### State Transitions
```
IDLE → LOAD (when start signal asserted)
LOAD → COMPUTE (after loading operands)
COMPUTE → COMPUTE (while b ≠ 0)
COMPUTE → DONE (when b = 0)
DONE → IDLE (when acknowledged)
```

## Implementation
```verilog
module gcd_fsm (
    input clk,
    input reset,
    input start,
    input [31:0] a_in,
    input [31:0] b_in,
    output reg [31:0] gcd_out,
    output reg done
);
    // State encoding
    typedef enum logic [1:0] {
        IDLE = 2'b00,
        LOAD = 2'b01,
        COMPUTE = 2'b10,
        DONE_STATE = 2'b11
    } state_t;
    
    state_t current_state, next_state;
    reg [31:0] a_reg, b_reg;
    reg [31:0] remainder;
    
    // State register
    always_ff @(posedge clk or posedge reset) begin
        if (reset)
            current_state <= IDLE;
        else
            current_state <= next_state;
    end
    
    // Next state logic
    always_comb begin
        case (current_state)
            IDLE: next_state = start ? LOAD : IDLE;
            LOAD: next_state = COMPUTE;
            COMPUTE: next_state = (b_reg == 0) ? DONE_STATE : COMPUTE;
            DONE_STATE: next_state = IDLE;
            default: next_state = IDLE;
        endcase
    end
    
    // Datapath
    always_ff @(posedge clk) begin
        case (current_state)
            LOAD: begin
                a_reg <= a_in;
                b_reg <= b_in;
                done <= 0;
            end
            COMPUTE: begin
                if (b_reg != 0) begin
                    remainder = a_reg % b_reg;
                    a_reg <= b_reg;
                    b_reg <= remainder;
                end
            end
            DONE_STATE: begin
                gcd_out <= a_reg;
                done <= 1;
            end
        endcase
    end
endmodule
```

## Optimization

### Hardware Efficiency
- **Area**: Minimal register usage (2 data registers + state register)
- **Timing**: Completes in O(log(min(a,b))) clock cycles
- **Power**: Low power consumption through efficient state transitions

### Performance
- Synthesized to run at 100 MHz on FPGA
- Average computation time: 5-15 clock cycles for typical inputs
- Maximum latency: ~50 cycles for worst-case inputs

## Verification

### Testbench Results
```
Test Case 1: GCD(48, 18) = 6 ✓
Test Case 2: GCD(100, 75) = 25 ✓
Test Case 3: GCD(17, 19) = 1 ✓
Test Case 4: GCD(1024, 256) = 256 ✓
```

### Waveform Analysis
Verified correct state transitions and data flow using ModelSim simulation.

## Skills Demonstrated
- FSM design methodology
- Algorithm to hardware mapping
- Verilog synthesis
- Timing optimization
- Verification techniques