---
title: "Custom 8-Bit Multicycle Processor"
date: 2024-11-16
draft: false
author: "William Lazcano"
tags:
  - SystemVerilog
  - FPGA
  - Processor Design
  - FSM
  - Digital Systems
image: /images/projects/processor.jpg
description: "Designed a complete 8-bit multicycle processor from scratch — custom ISA, FSM control unit, and full FPGA deployment on the Altera DE2-115."
toc: true
---

## Overview

This was the most ambitious project I've tackled so far: designing a working processor entirely from scratch. No starter code, no skeleton files — just a blank canvas, a specification, and SystemVerilog.

The result is a fully functional 8-bit multicycle CPU with a custom 16-instruction ISA, FSM-based control unit, and real deployment on an Altera DE2-115 FPGA. It successfully executes matrix operations in software using only integer arithmetic.

### Specs at a Glance

| Feature | Detail |
|---|---|
| Architecture | 8-bit multicycle |
| Instruction Set | 16 custom instructions |
| Memory | 512 bytes, byte-addressable |
| Cycles per Instruction | 1–4 depending on instruction type |
| Control Logic | FSM with 6 states |
| Target Platform | Altera DE2-115 FPGA |
| Language | SystemVerilog |

---

## Instruction Set Architecture

I designed all 16 instructions myself, grouped into four categories:

**Arithmetic:** `ADD`, `SUB`, `MULT`, `DIV`, `INAC` (increment accumulator)

**Logic:** `AND`, `OR`, `XOR`, `NOT`, `CLAC` (clear accumulator)

**Control Flow:** `JUMP`, `JMPZ` (jump if zero), `JPNZ` (jump if not zero)

**Data Movement:** `LDAC`, `STAC`, `MVAC`, `MOVR`

The architecture uses an accumulator model — most operations read from and write back to the accumulator (AC), which keeps the datapath simple and the control logic clean.

---

## FSM Control Unit

The control unit is a 6-state FSM that drives all control signals based on the current instruction and processor state.

```
FETCH_STATE → NORMAL_STATE
                 │
          ┌──────┼──────────────┐
          ▼      ▼              ▼
   LOAD_ADDRESS1  LOAD_STORE   JUMP_STATE
          │
          ▼
   LOAD_ADDRESS2
```

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
        fetch_state, normal_state,
        load_address1, load_address2,
        jump_state, load_store
    } FSM_states;

    FSM_states state, nextstate;
    reg Z;

    always@(posedge clk) begin
        if (reset || prog_mode)
            state <= fetch_state;
        else begin
            state <= nextstate;
            if(op_code[3]) Z <= FF;
        end
    end

    always@(*) begin
        case(state)
            fetch_state:
                nextstate <= normal_state;

            normal_state: begin
                if(op_code[3])
                    nextstate <= fetch_state;
                else if (op_code[3:0] == 4'b0001 || op_code[3:0] == 4'b0010 ||
                         op_code[3:0] == 4'b0101 || op_code[3:0] == 4'b0110 ||
                         op_code[3:0] == 4'b0111)
                    nextstate <= load_address1;
                else if (op_code[4])
                    nextstate <= load_store;
                else
                    nextstate <= fetch_state;
            end

            load_address1: nextstate <= load_address2;
            load_address2: nextstate <= (op_code[3:0] == 4'b0001 ||
                                         op_code[3:0] == 4'b0010) ?
                                         load_store : jump_state;
            jump_state:    nextstate <= fetch_state;
            load_store:    nextstate <= fetch_state;
            default:       nextstate <= fetch_state;
        endcase
    end
endmodule
```

---

## ALU

The ALU supports 8 operations selected by a 3-bit control signal and outputs a zero flag used by conditional jumps.

```systemverilog
module alu(
    input logic [7:0] a, b,
    input logic [2:0] alu_control,
    output logic [7:0] result,
    output logic zero_flag
);
    always_comb begin
        case(alu_control)
            3'b000: result = a + b;
            3'b001: result = a - b;
            3'b010: result = a + 1;   // INAC
            3'b011: result = 8'b0;    // CLAC
            3'b100: result = a & b;
            3'b101: result = a * b;
            3'b110: result = a / b;
            3'b111: result = ~a;
            default: result = 8'b0;
        endcase
        zero_flag = (result == 8'b0);
    end
endmodule
```

---

## Matrix Operations

To prove the processor actually works, I wrote a test program that performs 2×2 matrix addition, multiplication, and inversion entirely in software.

**Matrix Addition:**
```
A = | 6  5 |    B = | 1  2 |    C = | 7   7  |
    | 8  7 |        | 3  4 |        | 11  11 |
```

**Matrix Multiplication:**
```
A × B = | 21  32 |
        | 29  44 |
```

Since the processor has no floating-point support, matrix inversion uses fixed-point scaled integers — a deliberate design choice to demonstrate that complex algorithms are achievable in constrained hardware.

### Performance

| Operation | Clock Cycles | Time @ 50 MHz |
|---|---|---|
| Matrix Add (2×2) | ~120 | 2.4 μs |
| Matrix Multiply (2×2) | ~350 | 7.0 μs |
| Matrix Inversion (2×2) | ~500 | 10.0 μs |

---

## FPGA Deployment

Programming the processor onto real hardware added a layer of complexity beyond simulation. The DE2-115 board gives you 9 switches for address control, 8 for data, red LEDs for feedback, and 8 seven-segment displays showing the PC, opcode, and accumulator in real time.

```
Resource             Used     Available
Logic Elements       4,302    114,480
Dedicated Registers  4,171    114,480
Memory Bits          4,096    3,981,312
Utilization: ~3.8%
```

---

## Verification

Every instruction was verified in ModelSim before touching the FPGA. Waveform analysis confirmed correct state transitions, control signal generation, and memory operations across all test cases.

```
✓ ADD / SUB / MULT / DIV
✓ Matrix addition (2×2)
✓ Matrix multiplication (2×2)
✓ JMPZ / JPNZ conditional branching
✓ LDAC / STAC memory operations
```

---

## Challenges

**Integer matrix inversion** — no floating-point meant I had to implement fixed-point arithmetic with careful overflow management. It forced me to really understand how numerical precision works at the hardware level.

**512-byte memory constraint** — every byte mattered. I optimized instruction encoding aggressively to fit both the program and data in the same address space.

**Hardware debugging** — watching a processor fail silently on an FPGA is very different from a simulation. I built step-through debugging into the display output so I could pause execution and inspect state at each cycle.

---

## What I Learned

Building a processor from scratch makes every other digital design project easier. You stop thinking about components in isolation and start seeing the full picture — how control signals propagate, how datapaths interact, why timing matters. This project is the foundation everything else in my portfolio builds on.

---

**Code:** [GitHub](https://github.com/Will-L10)
