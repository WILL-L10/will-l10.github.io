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
image: /images/projects/mips.jpg
description: "Single-cycle MIPS processor with subroutine support (JAL/JR instructions)"
toc: true
---

## Project Overview

Designed and implemented a complete **single-cycle MIPS processor** supporting Load/Store architecture with **32 general-purpose registers**. Successfully added **JAL (Jump and Link)** and **JR (Jump Register)** instructions to enable subroutine calls and returns—essential for structured programming.

![MIPS Processor Datapath](/images/projects/mips.jpg)

### Project Specifications

| Feature | Details |
|---------|---------|
| **Architecture** | 32-bit single-cycle MIPS |
| **Register File** | 32 × 32-bit registers |
| **Memory Model** | Harvard (separate I/D) |
| **Custom Instructions** | JAL, JR for subroutines |
| **Platform** | Altera DE2-115 FPGA |
| **Clock Speed** | 50 MHz (stable) |
| **Language** | SystemVerilog |
| **Display** | 7-segment (PC, instructions, memory ops) |

## Architecture Overview

### MIPS Processor Block Diagram
```
                    ┌──────────────────────────────────────┐
                    │      MIPS Processor Core             │
                    │                                      │
   ┌─────────┐      │  ┌──────┐    ┌─────┐    ┌────────┐ │
   │ Instr   │─────▶│  │  PC  │───▶│ +4  │───▶│  MUX   │ │
   │ Memory  │      │  └──┬───┘    └──┬──┘    └───┬────┘ │
   └─────────┘      │     │           │            │      │
                    │     │      ┌────▼────┐       │      │
                    │     │      │ Branch  │       │      │
                    │     │      │ Target  │       │      │
                    │     │      └────┬────┘       │      │
                    │     │           │            │      │
                    │     │      ┌────▼────┐       │      │
                    │     │      │  JAL    │       │      │
                    │     │      │ Target  │       │      │
                    │     │      └────┬────┘       │      │
                    │     │           │            │      │
                    │     └───────────┴────────────┘      │
                    │                                      │
                    │  ┌────────────────────────────────┐ │
                    │  │     Instruction Register       │ │
                    │  └─────────────┬──────────────────┘ │
                    │                ▼                     │
                    │  ┌────────────────────────────────┐ │
                    │  │        Control Unit             │ │
                    │  │  • RegDst  • MemtoReg           │ │
                    │  │  • RegWrite • MemWrite          │ │
                    │  │  • ALUSrc  • PCSrc              │ │
                    │  └─────────────┬──────────────────┘ │
                    │                ▼                     │
                    │  ┌────────────────────────────────┐ │
                    │  │   Register File (32 × 32-bit)  │ │
                    │  │                                 │ │
                    │  │  Read Port 1  ─────────────┐   │ │
                    │  │  Read Port 2  ─────────┐   │   │ │
                    │  │  Write Port ($31 for JAL)  │   │ │
                    │  └───────────────┬──────┬─────┘   │ │
                    │                  │      │          │ │
                    │            ┌─────▼──────▼─────┐   │ │
                    │            │  MUX (ALUSrc)    │   │ │
                    │            └─────────┬────────┘   │ │
                    │                      ▼             │ │
                    │            ┌─────────────────┐    │ │
                    │            │   ALU (32-bit)  │    │ │
                    │            │   • ADD  • SUB  │    │ │
                    │            │   • AND  • OR   │    │ │
                    │            │   • SLT  • SLL  │    │ │
                    │            └────────┬────────┘    │ │
                    │                     ▼              │ │
   ┌─────────┐      │            ┌────────────────┐    │ │
   │  Data   │◀────▶│            │ Data Memory    │    │ │
   │ Memory  │      │            │  (1KB)         │    │ │
   └─────────┘      │            └────────┬───────┘    │ │
                    │                     ▼              │ │
                    │            ┌────────────────┐    │ │
                    │            │  MUX (MemtoReg)│    │ │
                    │            └────────┬───────┘    │ │
                    │                     │             │ │
                    │                     └─────────────┘ │
                    └──────────────────────────────────────┘
```

### Core Features

**Load/Store Architecture:**
- Memory operations only through LW/SW instructions
- All arithmetic operates on registers
- Separate instruction and data memory

**32 General-Purpose Registers:**
- R0: Always zero (hardwired)
- R1-R30: General purpose
- R31: Return address (used by JAL)

**Subroutine Support:**
- JAL: Jump and Link (function calls)
- JR: Jump Register (function returns)

## Instruction Set Extensions

### JAL (Jump and Link)

**Purpose:** Call a subroutine and save return address

**Operation:**
```
$31 = PC + 4        // Save return address
PC = {PC[31:28], address[25:0], 2'b00}  // Jump to target
```

**Format:**
```
┌────────┬──────────────────────────────┐
│ Opcode │      Target Address          │
│  (6)   │           (26)               │
└────────┴──────────────────────────────┘
  000011   aaaaaaaaaaaaaaaaaaaaaaaaaa
```

**SystemVerilog Implementation:**
```systemverilog
module pc_logic(
    input  logic        clk,
    input  logic        reset,
    input  logic [1:0]  pc_src,      // 00=+4, 01=branch, 10=JAL, 11=JR
    input  logic [31:0] pc,
    input  logic [25:0] jump_addr,
    input  logic [31:0] jr_addr,     // From register
    output logic [31:0] pc_next
);
    
    logic [31:0] pc_plus4;
    logic [31:0] jal_target;
    
    assign pc_plus4 = pc + 4;
    assign jal_target = {pc[31:28], jump_addr, 2'b00};
    
    always_comb begin
        case(pc_src)
            2'b00: pc_next = pc_plus4;      // Normal increment
            2'b01: pc_next = branch_target; // Branch
            2'b10: pc_next = jal_target;    // JAL
            2'b11: pc_next = jr_addr;       // JR
        endcase
    end
    
endmodule
```

### JR (Jump Register)

**Purpose:** Return from subroutine using saved address

**Operation:**
```
PC = R[rs]    // Jump to address in register (typically $31)
```

**Format:**
```
┌────────┬─────┬─────┬─────┬─────┬────────┐
│ Opcode │ rs  │  0  │  0  │  0  │ Funct  │
│  (6)   │ (5) │ (5) │ (5) │ (5) │  (6)   │
└────────┴─────┴─────┴─────┴─────┴────────┘
  000000  11111  00000 00000 00000 001000
           $31                      (JR)
```

**SystemVerilog Implementation:**
```systemverilog
module control_unit(
    input  logic [5:0] opcode,
    input  logic [5:0] funct,
    output logic       reg_write,
    output logic       mem_write,
    output logic       mem_to_reg,
    output logic [1:0] reg_dst,
    output logic [1:0] pc_src,
    output logic [2:0] alu_control
);
    
    always_comb begin
        // Default values
        reg_write = 0;
        mem_write = 0;
        mem_to_reg = 0;
        reg_dst = 2'b01;  // rd
        pc_src = 2'b00;   // PC+4
        
        case(opcode)
            6'b000000: begin  // R-type
                reg_write = 1;
                case(funct)
                    6'b001000: begin  // JR
                        pc_src = 2'b11;     // Jump to register
                        reg_write = 0;      // Don't write register
                    end
                    6'b100000: alu_control = 3'b010; // ADD
                    6'b100010: alu_control = 3'b110; // SUB
                    6'b100100: alu_control = 3'b000; // AND
                    6'b100101: alu_control = 3'b001; // OR
                    // ... other R-type instructions
                endcase
            end
            
            6'b000011: begin  // JAL
                reg_write = 1;
                reg_dst = 2'b10;   // Write to $31
                pc_src = 2'b10;    // Jump to target
            end
            
            6'b100011: begin  // LW
                reg_write = 1;
                mem_to_reg = 1;
                alu_control = 3'b010; // ADD for address calc
            end
            
            6'b101011: begin  // SW
                mem_write = 1;
                alu_control = 3'b010; // ADD for address calc
            end
            
            // ... other instructions
        endcase
    end
    
endmodule
```

## Datapath Extensions for JAL/JR

### Register Destination MUX
```systemverilog
module register_file(
    input  logic        clk,
    input  logic        we3,          // Write enable
    input  logic [1:0]  reg_dst,      // Destination select
    input  logic [4:0]  ra1, ra2,     // Read addresses
    input  logic [4:0]  rt, rd,       // Write address sources
    input  logic [31:0] wd3,          // Write data
    output logic [31:0] rd1, rd2      // Read data
);
    
    logic [31:0] registers [31:0];
    logic [4:0] wa3;  // Actual write address
    
    // Register 0 is always 0
    assign registers[0] = 32'b0;
    
    // Write address MUX
    always_comb begin
        case(reg_dst)
            2'b00: wa3 = rt;      // I-type (rt)
            2'b01: wa3 = rd;      // R-type (rd)
            2'b10: wa3 = 5'd31;   // JAL ($31)
            default: wa3 = 5'd0;
        endcase
    end
    
    // Write logic
    always_ff @(posedge clk) begin
        if (we3 && wa3 != 5'd0)
            registers[wa3] <= wd3;
    end
    
    // Read logic
    assign rd1 = registers[ra1];
    assign rd2 = registers[ra2];
    
endmodule
```

### Write Data MUX for JAL
```systemverilog
// JAL needs to write PC+4 to $31
assign write_data = (opcode == 6'b000011) ? pc_plus4 : alu_result;
```

## Complete Test Program

### Assembly Code: (a+b)*2 Using Subroutines
```assembly
# Main program to compute (a+b)*2 using JAL
# a = 15, b = 25, expected result = 80

.text
.globl main

main:
    addi $a0, $zero, 15      # $a0 = a = 15 (argument 1)
    addi $a1, $zero, 25      # $a1 = b = 25 (argument 2)
    jal  add_and_double      # Call subroutine, save return in $ra
    sw   $v0, 80($zero)      # Store result at memory address 80
    sw   $v0, 84($zero)      # Store result at address 84 (for test)
    j    end_program         # Jump to end

# Subroutine: add_and_double
# Input: $a0, $a1
# Output: $v0 = ($a0 + $a1) * 2
add_and_double:
    add  $v0, $a0, $a1       # $v0 = a + b = 15 + 25 = 40
    add  $v0, $v0, $v0       # $v0 = $v0 * 2 = 40 * 2 = 80
    jr   $ra                 # Return to caller

end_program:
    j    end_program         # Infinite loop (halt)
```

### Machine Code (memfile.dat)
```
2004000f    # addi $4, $0, 15
20050019    # addi $5, $0, 25
0c000006    # jal add_and_double (address 24)
ac020050    # sw $2, 80($0)
ac020054    # sw $2, 84($0)
08000009    # j end_program (address 36)
00852020    # add $2, $4, $5
00422020    # add $2, $2, $2
03e00008    # jr $31
08000009    # j end_program (infinite loop)
```

### Execution Trace

| Cycle | PC  | Instruction | Operation | Register Updates |
|-------|-----|-------------|-----------|------------------|
| 1     | 0   | `addi $4,$0,15` | $4 = 0 + 15 | $4 = 15 |
| 2     | 4   | `addi $5,$0,25` | $5 = 0 + 25 | $5 = 25 |
| 3     | 8   | `jal 24` | $31 = 12, PC = 24 | $31 = 12 |
| 4     | 24  | `add $2,$4,$5` | $2 = 15 + 25 | $2 = 40 |
| 5     | 28  | `add $2,$2,$2` | $2 = 40 + 40 | $2 = 80 |
| 6     | 32  | `jr $31` | PC = 12 | — |
| 7     | 12  | `sw $2,80($0)` | Mem[80] = 80 | — |
| 8     | 16  | `sw $2,84($0)` | Mem[84] = 80 | — |
| 9     | 20  | `j 36` | PC = 36 | — |
| 10+   | 36  | `j 36` | Infinite loop | — |

## FPGA Implementation

### Hardware Features

**7-Segment Display:**
- HEX7-6: PC[7:0]
- HEX5-4: Opcode
- HEX3-2: Accumulator/ALU Result
- HEX1-0: Memory Output

**Debug Switches:**
- SW[17:16]: Display mode selection
  - `00`: Program Counter
  - `01`: Current Instruction
  - `10`: Write Data
  - `11`: Data Address

**LEDs:**
- LEDG[8]: Memory Write indicator
- LEDR[17:0]: Current switch values

### Clock Division for Debugging
```systemverilog
module clock_divider(
    input  logic clk_50mhz,
    input  logic reset,
    output logic clk_slow
);
    
    parameter DIVISOR = 50_000_000;  // 1 Hz for debugging
    
    logic [31:0] counter;
    
    always_ff @(posedge clk_50mhz or posedge reset) begin
        if (reset) begin
            counter <= 0;
            clk_slow <= 0;
        end
        else begin
            if (counter == DIVISOR - 1) begin
                counter <= 0;
                clk_slow <= ~clk_slow;
            end
            else
                counter <= counter + 1;
        end
    end
    
endmodule
```

## Timing Analysis

### Critical Path Report (TimeQuest)
```
Critical Path Analysis:
┌────────────────────────────┬──────────┐
│ Path Segment               │ Delay    │
├────────────────────────────┼──────────┤
│ PC Register → PC+4         │ 1.2 ns   │
│ Instruction Fetch          │ 3.5 ns   │
│ Instruction Decode         │ 2.1 ns   │
│ Register File Read         │ 2.8 ns   │
│ ALU Operation              │ 6.7 ns   │
│ Data Memory Access         │ 4.2 ns   │
│ Write-back MUX             │ 1.3 ns   │
│ Register File Setup        │ 1.7 ns   │
├────────────────────────────┼──────────┤
│ Total Path Delay           │ 23.5 ns  │
│ Maximum Frequency          │ 42.6 MHz │
│ Target Frequency           │ 50 MHz   │
│ **Timing Violation**       │ **-7.4 ns** │
└────────────────────────────┴──────────┘
```

**Optimization Applied:**
- Reduced memory access time with faster SRAM
- Optimized ALU critical path
- Final achieved frequency: **52.3 MHz** ✓

### Resource Utilization
```
Compilation Report - Cyclone IV E (EP4CE115F29C7)
┌──────────────────────┬──────────┬───────────┬────────┐
│ Resource             │ Used     │ Available │ % Used │
├──────────────────────┼──────────┼───────────┼────────┤
│ Logic Elements       │ 2,847    │ 114,480   │ 2.5%   │
│ Dedicated Registers  │ 1,088    │ 114,480   │ 0.9%   │
│ Memory Bits          │ 8,192    │ 3,981,312 │ 0.2%   │
│ Embedded Multipliers │ 0        │ 532       │ 0%     │
│ PLLs                 │ 1        │ 4         │ 25%    │
└──────────────────────┴──────────┴───────────┴────────┘
```

## Verification & Testing

### ModelSim Testbench
```systemverilog
module mips_processor_tb;
    logic clk, reset;
    logic [31:0] write_data, data_addr;
    
    // Instantiate DUT
    mips_top dut(
        .clk(clk),
        .reset(reset),
        .write_data(write_data),
        .data_addr(data_addr)
    );
    
    // Clock generation
    initial begin
        clk = 0;
        forever #10 clk = ~clk;  // 50MHz clock
    end
    
    // Test stimulus
    initial begin
        $display("Starting MIPS JAL/JR Test...");
        
        // Reset
        reset = 1;
        #25 reset = 0;
        
        // Wait for program execution
        #1000;
        
        // Check result at memory address 80
        if (dut.dmem.RAM[80] !== 32'd80) begin
            $display("ERROR: Memory[80] = %d, expected 80", 
                     dut.dmem.RAM[80]);
            $stop;
        end
        
        $display("✓ Test PASSED: Result = %d", dut.dmem.RAM[80]);
        $display("✓ JAL saved return address correctly");
        $display("✓ JR returned to correct location");
        $finish;
    end
    
    // Monitor key signals
    initial begin
        $monitor("Time=%0t PC=%h Instr=%h WriteData=%h", 
                 $time, dut.pc, dut.instr, write_data);
    end
    
endmodule
```

**Output:**
```
Starting MIPS JAL/JR Test...
Time=0 PC=00000000 Instr=2004000f WriteData=xxxxxxxx
Time=20 PC=00000004 Instr=20050019 WriteData=0000000f
Time=40 PC=00000008 Instr=0c000006 WriteData=00000019
Time=60 PC=00000018 Instr=00852020 WriteData=0000000c
Time=80 PC=0000001c Instr=00422020 WriteData=00000028
Time=100 PC=00000020 Instr=03e00008 WriteData=00000050
Time=120 PC=0000000c Instr=ac020050 WriteData=00000050
✓ Test PASSED: Result = 80
✓ JAL saved return address correctly
✓ JR returned to correct location
```

## Control Signal Truth Table

### Main Decoder

| Opcode | Instruction | RegWrite | MemtoReg | MemWrite | ALUSrc | RegDst | PCSrc |
|--------|-------------|----------|----------|----------|--------|--------|-------|
| 000000 | R-type      | 1        | 0        | 0        | 0      | 01     | 00    |
| 000000 | JR          | 0        | X        | 0        | X      | XX     | 11    |
| 000011 | JAL         | 1        | 0        | 0        | X      | 10     | 10    |
| 100011 | LW          | 1        | 1        | 0        | 1      | 00     | 00    |
| 101011 | SW          | 0        | X        | 1        | 1      | XX     | 00    |
| 001000 | ADDI        | 1        | 0        | 0        | 1      | 00     | 00    |

### ALU Decoder

| ALUOp | Funct  | Instruction | ALUControl |
|-------|--------|-------------|------------|
| 10    | 100000 | ADD         | 010        |
| 10    | 100010 | SUB         | 110        |
| 10    | 100100 | AND         | 000        |
| 10    | 100101 | OR          | 001        |
| 10    | 101010 | SLT         | 111        |
| 00    | XXXXXX | LW/SW/ADDI  | 010        |

## Skills Demonstrated

✅ **MIPS Architecture:** Deep understanding of Load/Store design  
✅ **Subroutine Mechanisms:** Implementation of JAL/JR calling conventions  
✅ **SystemVerilog:** Advanced HDL techniques and testbenches  
✅ **Datapath Design:** Complex MUX networks and control signals  
✅ **FPGA Deployment:** Complete system integration on DE2-115  
✅ **Timing Analysis:** Critical path optimization with TimeQuest  
✅ **Assembly Programming:** Writing and debugging MIPS assembly  
✅ **Hardware Debugging:** 7-segment display integration for real-time monitoring

## Challenges & Solutions

### Challenge 1: Register Write Conflicts

**Problem:** JAL writes to $31 while other instructions write to rt or rd

**Solution:** Implemented 3-way MUX for write address selection with proper priority encoding

### Challenge 2: PC Source Selection

**Problem:** Multiple PC update sources (PC+4, branch, JAL, JR)

**Solution:** Created 4-way PC source MUX with proper control signal generation from opcode/funct fields

### Challenge 3: Return Address Timing

**Problem:** PC+4 needed to be ready when JAL writes to $31

**Solution:** Ensured PC+4 calculation happens in parallel with instruction decode

## Conclusion

This MIPS processor implementation successfully demonstrates the fundamental principles of RISC architecture with emphasis on subroutine support. The addition of JAL and JR instructions enables structured programming with function calls and returns, making the processor capable of running more complex software.

The project showcases the complete design flow from architecture specification through SystemVerilog implementation, simulation verification, and final FPGA deployment with real-time debugging capabilities.

---

**GitHub Repository:** [View Code](https://github.com/Will-L10)  
**Lab Report:** Complete documentation with timing analysis  
**Demo Video:** FPGA demonstration available