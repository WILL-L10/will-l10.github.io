---
title: "MIPS Processor with Subroutine Support"
date: 2024-09-20
draft: false
author: "William Lazcano"
tags:
  - MIPS
  - FPGA
  - SystemVerilog
  - Digital Design
image: /images/projects/mips.jpg
description: "Single-cycle MIPS processor extended with JAL and JR instructions to enable full subroutine call and return support."
toc: true
---

## Overview

This project builds on a standard single-cycle MIPS processor by adding **JAL** (Jump and Link) and **JR** (Jump Register) — the two instructions that make subroutine calls possible in MIPS. Without them, you can write loops but not functions. With them, you have the foundation for structured software.

The processor was implemented in SystemVerilog, tested with a real assembly program, and deployed on the Altera DE2-115 FPGA with 7-segment display output for live debugging.

### Specs at a Glance

| Feature | Detail |
|---|---|
| Architecture | 32-bit single-cycle MIPS |
| Register File | 32 × 32-bit (R0–R31) |
| Custom Instructions | JAL, JR |
| Memory Model | Harvard (separate I/D) |
| Platform | Altera DE2-115 FPGA |
| Clock Speed | 50 MHz |
| Language | SystemVerilog |

---

## JAL — Jump and Link

JAL is the function call instruction in MIPS. It saves the return address in `$31` and jumps to the target.

```
$31 = PC + 4
PC  = {PC[31:28], instr[25:0], 2'b00}
```

```systemverilog
module pc_logic(
    input  logic        clk, reset,
    input  logic [1:0]  pc_src,
    input  logic [31:0] pc,
    input  logic [25:0] jump_addr,
    input  logic [31:0] jr_addr,
    output logic [31:0] pc_next
);
    logic [31:0] pc_plus4, jal_target;

    assign pc_plus4   = pc + 4;
    assign jal_target = {pc[31:28], jump_addr, 2'b00};

    always_comb begin
        case(pc_src)
            2'b00: pc_next = pc_plus4;
            2'b01: pc_next = branch_target;
            2'b10: pc_next = jal_target;
            2'b11: pc_next = jr_addr;
        endcase
    end
endmodule
```

---

## JR — Jump Register

JR returns from a subroutine by jumping to the address stored in a register (typically `$31`).

```
PC = R[rs]
```

The key implementation challenge is routing the register file read output back to the PC logic. A 4-way PC source MUX handles all possible next-PC values: PC+4, branch target, JAL target, and JR register value.

---

## Register File — 3-Way Write Address MUX

JAL writes to `$31` instead of `rt` or `rd`. This requires a 3-way MUX for the write address:

```systemverilog
always_comb begin
    case(reg_dst)
        2'b00: wa3 = rt;      // I-type
        2'b01: wa3 = rd;      // R-type
        2'b10: wa3 = 5'd31;   // JAL
        default: wa3 = 5'd0;
    endcase
end
```

JAL also needs to write `PC+4` (not ALU result) as the return address:

```systemverilog
assign write_data = (opcode == 6'b000011) ? pc_plus4 : alu_result;
```

---

## Control Unit

```systemverilog
module control_unit(
    input  logic [5:0] opcode, funct,
    output logic       reg_write, mem_write, mem_to_reg,
    output logic [1:0] reg_dst, pc_src,
    output logic [2:0] alu_control
);
    always_comb begin
        reg_write  = 0; mem_write = 0;
        mem_to_reg = 0; reg_dst   = 2'b01;
        pc_src     = 2'b00;

        case(opcode)
            6'b000000: begin  // R-type
                reg_write = 1;
                case(funct)
                    6'b001000: begin  // JR
                        pc_src    = 2'b11;
                        reg_write = 0;
                    end
                    6'b100000: alu_control = 3'b010; // ADD
                    6'b100010: alu_control = 3'b110; // SUB
                    6'b100100: alu_control = 3'b000; // AND
                    6'b100101: alu_control = 3'b001; // OR
                endcase
            end
            6'b000011: begin  // JAL
                reg_write = 1;
                reg_dst   = 2'b10;
                pc_src    = 2'b10;
            end
            6'b100011: begin  // LW
                reg_write  = 1; mem_to_reg = 1;
                alu_control = 3'b010;
            end
            6'b101011: begin  // SW
                mem_write   = 1;
                alu_control = 3'b010;
            end
        endcase
    end
endmodule
```

---

## Test Program: Computing (a + b) × 2 with a Subroutine

To verify JAL and JR actually work together, I wrote an assembly program that calls a subroutine and verifies the result in data memory.

```assembly
main:
    addi $a0, $zero, 15       # a = 15
    addi $a1, $zero, 25       # b = 25
    jal  add_and_double        # call subroutine, $ra = PC+4
    sw   $v0, 80($zero)        # store result
    j    end_program

add_and_double:
    add  $v0, $a0, $a1         # $v0 = 15 + 25 = 40
    add  $v0, $v0, $v0         # $v0 = 40 * 2  = 80
    jr   $ra                   # return

end_program:
    j    end_program
```

### Execution Trace

| Cycle | PC | Instruction | Key Operation |
|---|---|---|---|
| 1 | 0 | addi $4,$0,15 | $4 = 15 |
| 2 | 4 | addi $5,$0,25 | $5 = 25 |
| 3 | 8 | jal 24 | $31 = 12, PC = 24 |
| 4 | 24 | add $2,$4,$5 | $2 = 40 |
| 5 | 28 | add $2,$2,$2 | $2 = 80 |
| 6 | 32 | jr $31 | PC = 12 |
| 7 | 12 | sw $2,80($0) | Mem[80] = 80 |

---

## Verification

```systemverilog
initial begin
    reset = 1; #25 reset = 0;
    #1000;

    if (dut.dmem.RAM[80] !== 32'd80)
        $display("FAIL: got %d", dut.dmem.RAM[80]);
    else begin
        $display("✓ Result correct: %d", dut.dmem.RAM[80]);
        $display("✓ JAL saved return address correctly");
        $display("✓ JR returned to correct location");
    end
end
```

**Output:**
```
✓ Result correct: 80
✓ JAL saved return address correctly
✓ JR returned to correct location
```

---

## Timing & Resource Utilization

```
Critical Path:          23.5 ns → Initial: 42.6 MHz
After optimization:     19.1 ns → 52.3 MHz ✓

Resource Usage (Cyclone IV E):
  Logic Elements       2,847 / 114,480   (2.5%)
  Dedicated Registers  1,088 / 114,480
  Memory Bits          8,192 / 3,981,312
  PLLs                 1 / 4
```

---

## Control Signal Truth Table

| Opcode | Instruction | RegWrite | MemtoReg | RegDst | PCSrc |
|---|---|---|---|---|---|
| 000000 | R-type | 1 | 0 | 01 | 00 |
| 000000 | JR | 0 | X | XX | 11 |
| 000011 | JAL | 1 | 0 | 10 | 10 |
| 100011 | LW | 1 | 1 | 00 | 00 |
| 101011 | SW | 0 | X | XX | 00 |

---

## What I Learned

Subroutine support looks simple on paper — two instructions — but implementing it correctly required thinking through every datapath interaction at once: where the return address comes from, where it goes, how the PC source gets selected, and why the register write address needs a 3-way MUX. Getting the first `jr $ra` to actually return to the right place was genuinely satisfying.

---

**Code:** [GitHub](https://github.com/Will-L10)
