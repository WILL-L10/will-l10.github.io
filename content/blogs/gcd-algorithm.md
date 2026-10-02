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
image: /images/projects/gcd-rtl-cover.jpg
description: "Implemented the Euclidean GCD algorithm in hardware using a finite state machine — deployed on FPGA with real-time switch input and 7-segment output."
toc: true
weight: 5
category: "Digital design and timing"
summary: "Turned Euclid's GCD algorithm into a hardware datapath and FSM controller, then debugged a real timing violation in TimeQuest."
images: ["/images/projects/gcd-rtl-cover.jpg"]
---

## Overview

This project takes a classic algorithm — Greatest Common Divisor using the Euclidean method — and implements it entirely in hardware. No processor, no software. Just a finite state machine driving a datapath, computing GCD directly in logic.

It's a foundational exercise in the difference between software and hardware design: in software, you write a loop; in hardware, you design state machines and datapaths that do the same work in a completely different way.

### Specs at a Glance

| Feature | Detail |
|---|---|
| Algorithm | Euclidean subtraction method |
| Data Width | 5-bit inputs (`a_in[4:0]`, `b_in[4:0]`) |
| Architecture | FSM-based datapath |
| Input Range | 1–31 |
| Platform | Altera DE2-115 FPGA |
| Clock Speed | 50 MHz |
| Output | Result on two seven-segment displays (HEX1–HEX0) |
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

**Example — GCD(24, 18):**

| Step | a | b | Operation |
|---|---|---|---|
| 0 | 24 | 18 | a > b → a = 6 |
| 1 | 6 | 18 | a < b → b = 12 |
| 2 | 6 | 12 | a < b → b = 6 |
| 3 | 6 | 6 | a = b → **GCD = 6** |

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

## Synthesized Design

Quartus's RTL viewer shows the clean split between the controller and the datapath: the FSM only produces control signals (`load_a`, `load_b`, `sub_a`, `sub_b`), and the datapath feeds back status flags (`eq`, `gt`).

<figure>
  <img src="/images/projects/gcd-rtl.png" alt="Quartus RTL view of the GCD controller and datapath" style="max-width:100%;height:auto;border-radius:6px" loading="lazy">
  <figcaption style="font-size:0.9em;opacity:0.8;margin-top:6px">RTL view of the synthesized design: <code>control:ctrl</code> drives <code>datapath:dp</code>, whose result goes to two <code>hex_display</code> decoders.</figcaption>
</figure>

---

## Verilog Implementation

```verilog
module gcd_calculator(
    input  wire       clk, reset, start,
    input  wire [4:0] a_in, b_in,
    output reg  [4:0] gcd_out,
    output reg        done
);
    localparam IDLE=3'b000, LOAD=3'b001, COMP=3'b010,
               SUB_A=3'b011, SUB_B=3'b100, DONE=3'b101;

    reg [2:0] state, next_state;
    reg [4:0] a, b, next_a, next_b;

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

- Switches — Inputs A and B (5 bits each)
- `KEY` — Start and reset
- `HEX1–HEX0` — GCD result in hex

---

## Example Runs

The subtraction method takes one compare/subtract iteration per step, so latency depends on the inputs:

```
GCD(24, 18) = 6    3 subtract steps
GCD(21, 14) = 7    2 subtract steps
GCD(17, 13) = 1    7 subtract steps
GCD(31, 30) = 1   30 subtract steps   ← worst case: one input is much larger than the difference
GCD(20, 20) = 20   0 subtract steps
```

---

## Timing Analysis: Finding and Fixing a Violation

This lab was also my first real timing problem. TimeQuest reported **10 failing setup paths with a worst-case slack of −2.956 ns**, all inside my clock-divider counter, even though simulation passed.

<figure>
  <img src="/images/projects/gcd-timing-fail.png" alt="TimeQuest report showing negative setup slack" style="max-width:100%;height:auto;border-radius:6px" loading="lazy">
  <figcaption style="font-size:0.9em;opacity:0.8;margin-top:6px">Before: TimeQuest flags the <code>clock_divider</code> counter paths in red, with worst setup slack −2.956 ns.</figcaption>
</figure>

Reading the console log revealed the real cause: there was **no clock constraint**, so TimeQuest fell back to `derive_clocks -period 1.0` and analyzed the design as if it ran at 1 GHz. I added the proper constraint for the board's 50 MHz oscillator (`create_clock -period 20.000 -name CLOCK_50`) and re-ran the analysis:

<figure>
  <img src="/images/projects/gcd-timing-pass.png" alt="TimeQuest report after fixing the timing violation" style="max-width:100%;height:auto;border-radius:6px" loading="lazy">
  <figcaption style="font-size:0.9em;opacity:0.8;margin-top:6px">After: 0 violated paths, with worst-case setup slack of <b>+0.291 ns</b> at 50 MHz.</figcaption>
</figure>

I also built a 16×8 single-port RAM, initialized from a memory file, to practice synchronous read/write timing:

<figure>
  <img src="/images/projects/gcd-ram-rtl.png" alt="RTL view of the single-port RAM with debouncer and hex decoders" style="max-width:100%;height:auto;border-radius:6px" loading="lazy">
  <figcaption style="font-size:0.9em;opacity:0.8;margin-top:6px">RAM test system: button debouncer → single-port RAM → four hex decoders driving the seven-segment displays.</figcaption>
</figure>

<figure>
  <img src="/images/projects/gcd-ram-board.jpg" alt="DE2-115 running the RAM test design" style="max-width:100%;height:auto;border-radius:6px" loading="lazy">
  <figcaption style="font-size:0.9em;opacity:0.8;margin-top:6px">The RAM design running on the DE2-115.</figcaption>
</figure>

---

## Hardware vs. Software

| | Software (Python) | Hardware (Verilog) |
|---|---|---|
| Execution | Loop on a general-purpose CPU | One subtract per clock cycle |
| Latency | Variable, OS-dependent | Fully deterministic |
| Flexibility | Easy to modify | Fixed after synthesis |
| Resource use | Entire CPU | A few registers, a comparator, and a subtractor |

The hardware wins on determinism. Every call to GCD with the same inputs takes exactly the same number of clock cycles — every single time. That matters in real-time systems.

---

## What I Learned

This project made the software-to-hardware translation click for me. Writing a while loop is trivial. Designing the FSM that replicates it — thinking through every state, every transition, every control signal — forces you to understand what that loop is actually doing at every step. It also introduced me to the reality of hardware debugging: a timing issue in COMP→SUB_A showed up as an intermittent wrong answer that only failed on certain input patterns. Waveform analysis in ModelSim was the only way to catch it.

---

**Code:** [GitHub](https://github.com/Will-L10)
