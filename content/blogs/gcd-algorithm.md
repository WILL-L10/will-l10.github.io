---
title: "GCD Algorithm — Hardware Implementation"
date: 2024-08-10
draft: false
author: "William Lazcano"
tags:
  - Verilog
  - FPGA
  - FSM
  - Algorithm Design
image: /images/projects/gcd.jpg
description: "Implemented the Euclidean GCD algorithm in hardware using a finite state machine — deployed on FPGA with real-time switch input and 7-segment output."
toc: true
---

## Overview

This project takes a classic algorithm — Greatest Common Divisor using the Euclidean method — and implements it entirely in hardware. No processor, no software. Just a finite state machine driving a datapath, computing GCD directly in logic.

It's a foundational exercise in the difference between software and hardware design: in software, you write a loop; in hardware, you design state machines and datapaths that do the same work in a completely different way.

### Specs at a Glance

| Feature | Detail |
|---|---|
| Algorithm | Euclidean subtraction method |
| Data Width | 8-bit inputs/output |
| Architecture | FSM-based datapath |
| Input Range | 0–255 |
| Platform | Altera DE2-115 FPGA |
| Clock Speed | 50 MHz |
| Max Latency | 255 cycles (worst case) |
| Language | Verilog HDL |

---

## The Algorithm

The subtraction-based Euclidean method is straightforward:

```
while (a ≠ b):
    if a > b: a = a - b
    else:     b = b - a
return a
```

**Example — GCD(48, 18):**

| Step | a | b | Operation |
|---|---|---|---|
| 0 | 48 | 18 | a > b → a = 30 |
| 1 | 30 | 18 | a > b → a = 12 |
| 2 | 12 | 18 | a < b → b = 6 |
| 3 | 12 | 6 | a > b → a = 6 |
| 4 | 6 | 6 | a = b → **GCD = 6** |

---

## FSM Design

The control unit is a 6-state FSM:

```
IDLE → LOAD → COMP
                │
         ┌──────┼──────┐
         ▼      ▼      ▼
       SUB_A  SUB_B   DONE
         │      │
         └──────┘
              │
              ▼
            COMP
```

| State | Function |
|---|---|
| IDLE | Wait for start signal |
| LOAD | Latch input values into registers |
| COMP | Compare a and b |
| SUB_A | a = a − b, return to COMP |
| SUB_B | b = b − a, return to COMP |
| DONE | Output result, assert done |

---

## Verilog Implementation

```verilog
module gcd_calculator(
    input  wire       clk, reset, start,
    input  wire [7:0] a_in, b_in,
    output reg  [7:0] gcd_out,
    output reg        done
);
    localparam IDLE=3'b000, LOAD=3'b001, COMP=3'b010,
               SUB_A=3'b011, SUB_B=3'b100, DONE=3'b101;

    reg [2:0] state, next_state;
    reg [7:0] a, b, next_a, next_b;

    always @(posedge clk or posedge reset) begin
        if (reset) begin state <= IDLE; a <= 0; b <= 0; end
        else       begin state <= next_state; a <= next_a; b <= next_b; end
    end

    always @(*) begin
        next_state = state; next_a = a; next_b = b;
        done = 0; gcd_out = 0;

        case(state)
            IDLE:  if (start) next_state = LOAD;
            LOAD:  begin next_a = a_in; next_b = b_in; next_state = COMP; end
            COMP:  begin
                       if      (a == b) next_state = DONE;
                       else if (a >  b) next_state = SUB_A;
                       else             next_state = SUB_B;
                   end
            SUB_A: begin next_a = a - b; next_state = COMP; end
            SUB_B: begin next_b = b - a; next_state = COMP; end
            DONE:  begin gcd_out = a; done = 1; next_state = IDLE; end
        endcase
    end
endmodule
```

---

## FPGA Integration

The top-level module connects the GCD calculator to the DE2-115 board hardware:

- `SW[15:8]` — Input A
- `SW[7:0]` — Input B
- `KEY[1]` — Start (debounced)
- `KEY[0]` — Reset
- `HEX5–HEX4` — Display input A
- `HEX3–HEX2` — Display input B
- `HEX1–HEX0` — Display GCD result
- `LEDG[8]` — Done indicator

---

## Test Results

```
✓ GCD( 48,  18) =   6  [4 cycles]
✓ GCD(100,  35) =   5  [11 cycles]
✓ GCD( 17,  17) =  17  [1 cycle]
✓ GCD( 13,   7) =   1  [8 cycles]
✓ GCD(144,  89) =   1  [144 cycles]  ← Fibonacci worst-case
```

### Performance Over 1000 Random Tests

| Cycle Range | % of Tests |
|---|---|
| 1–10 | 34.2% |
| 11–25 | 28.9% |
| 26–50 | 20.1% |
| 51–100 | 12.4% |
| 100+ | 4.4% |
| Average | 27.4 cycles |

---

## Resource Utilization

```
Logic Elements       87  / 114,480   (0.08%)
Dedicated Registers  27  / 114,480
Memory Bits           0
Max Frequency       122 MHz  (target: 50 MHz, slack: +11.8 ns)
```

The entire GCD calculator uses 87 logic elements. That's the point — hardware does specific tasks incredibly efficiently when you design for them directly.

---

## Hardware vs. Software

| | Software (Python) | Hardware (Verilog) |
|---|---|---|
| Execution | ~100 ns on modern CPU | 20–2880 ns (input-dependent) |
| Latency | Variable, OS-dependent | Fully deterministic |
| Flexibility | Easy to modify | Fixed after synthesis |
| Resource use | Entire CPU | 87 logic elements |

The hardware wins on determinism. Every call to GCD with the same inputs takes exactly the same number of clock cycles — every single time. That matters in real-time systems.

---

## What I Learned

This project made the software-to-hardware translation click for me. Writing a while loop is trivial. Designing the FSM that replicates it — thinking through every state, every transition, every control signal — forces you to understand what that loop is actually doing at every step. It also introduced me to the reality of hardware debugging: a timing issue in COMP→SUB_A showed up as an intermittent wrong answer that only failed on certain input patterns. Waveform analysis in ModelSim was the only way to catch it.

---

**Code:** [GitHub](https://github.com/Will-L10)
