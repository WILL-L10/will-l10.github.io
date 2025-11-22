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
description: "5-stage pipelined MIPS processor with hazard detection and forwarding"
toc: true
---

## Project Overview

Designed and implemented a complete single-cycle MIPS processor with subroutine support, demonstrating understanding of Load/Store architecture and function call mechanisms.

## Architecture

### Core Features
- **32 General-Purpose Registers**: Full MIPS register file
- **Load/Store Architecture**: Separate instructions for memory access and computation
- **Subroutine Support**: JAL and JR instructions for function calls
- **Harvard Architecture**: Separate instruction and data memory

### Extended Instructions
Added critical instructions for structured programming:
- **JAL (Jump and Link)**: Save return address and jump to function
  - Stores PC+4 in register $31 (return address register)
  - Jumps to target address
  
- **JR (Jump Register)**: Return from function
  - Jumps to address in specified register
  - Essential for function returns

## Implementation Details

### Datapath Extensions
```systemverilog
module mips_processor (
    input logic clk,
    input logic reset,
    output logic [31:0] pc_out,
    output logic [31:0] instr_out
);
    // Register file with 32 registers
    logic [31:0] registers [31:0];
    logic [31:0] pc;
    logic [31:0] instr;
    
    // Control signals
    logic reg_write, mem_write, mem_to_reg;
    logic [1:0] reg_dst;  // For selecting destination register
    logic [1:0] pc_src;   // For JAL/JR control
    
    // ALU
    logic [31:0] alu_result;
    logic [31:0] src_a, src_b;
    
    // Data memory
    logic [31:0] read_data;
    logic [31:0] write_data;
    
    // PC Logic with JAL/JR support
    always_ff @(posedge clk or posedge reset) begin
        if (reset)
            pc <= 32'b0;
        else begin
            case (pc_src)
                2'b00: pc <= pc + 4;              // Normal increment
                2'b01: pc <= {pc[31:28], instr[25:0], 2'b00};  // JAL
                2'b10: pc <= registers[instr[25:21]];  // JR
                default: pc <= pc + 4;
            endcase
        end
    end
    
    // Register file with write-back MUX
    always_ff @(posedge clk) begin
        if (reg_write) begin
            case (reg_dst)
                2'b00: registers[instr[20:16]] <= write_data;  // rt
                2'b01: registers[instr[15:11]] <= write_data;  // rd
                2'b10: registers[31] <= pc + 4;  // $ra for JAL
            endcase
        end
    end
    
    // MUX for write data routing
    assign write_data = mem_to_reg ? read_data : alu_result;
    
endmodule
```

### Control Unit
```systemverilog
module control_unit (
    input logic [5:0] opcode,
    input logic [5:0] funct,
    output logic reg_write,
    output logic mem_write,
    output logic mem_to_reg,
    output logic [1:0] reg_dst,
    output logic [1:0] pc_src,
    output logic [2:0] alu_control
);
    always_comb begin
        // Default values
        reg_write = 0;
        mem_write = 0;
        mem_to_reg = 0;
        reg_dst = 2'b01;
        pc_src = 2'b00;
        
        case (opcode)
            6'b000000: begin  // R-type
                reg_write = 1;
                case (funct)
                    6'b001000: begin  // JR
                        pc_src = 2'b10;
                        reg_write = 0;
                    end
                    // Other R-type instructions...
                endcase
            end
            
            6'b000011: begin  // JAL
                reg_write = 1;
                reg_dst = 2'b10;   // Write to $31
                pc_src = 2'b01;    // Jump
            end
            
            // Other instructions...
        endcase
    end
endmodule
```

## Assembly Programming

### Test Program: Function Call Demo
```assembly
# Main program
main:
    addi $a0, $zero, 5     # a = 5
    addi $a1, $zero, 3     # b = 3
    jal  compute           # Call compute(a, b)
    # Result now in $v0
    j    end               # Jump to end

# Function: compute (a+b)*2
compute:
    add  $t0, $a0, $a1     # t0 = a + b
    sll  $v0, $t0, 1       # v0 = t0 * 2 (shift left)
    jr   $ra               # Return

end:
    # Program complete
```

## Hardware Deployment

### FPGA Implementation
- **Platform**: Altera DE2-115
- **Clock**: 50 MHz with clock division
- **Display**: 7-segment showing PC, instructions, memory operations
- **Debug Features**: 
  - Pause-on-write capability
  - Step-by-step execution
  - Real-time register display

### Memory Configuration
```systemverilog
// Instruction Memory (1KB)
module instruction_memory (
    input logic [31:0] addr,
    output logic [31:0] instr
);
    logic [31:0] mem [0:255];
    
    initial begin
        $readmemh("program.hex", mem);
    end
    
    assign instr = mem[addr[31:2]];
endmodule

// Data Memory (1KB)
module data_memory (
    input logic clk,
    input logic mem_write,
    input logic [31:0] addr,
    input logic [31:0] write_data,
    output logic [31:0] read_data
);
    logic [31:0] mem [0:255];
    
    always_ff @(posedge clk) begin
        if (mem_write)
            mem[addr[31:2]] <= write_data;
    end
    
    assign read_data = mem[addr[31:2]];
endmodule
```

## Testing & Verification

### Test Cases
1. **Basic arithmetic**: ADD, SUB, AND, OR
2. **Memory operations**: LW, SW with various offsets
3. **Function calls**: JAL/JR with nested calls
4. **Edge cases**: Writing to $zero, jumping to boundary addresses

### Timing Analysis
```
Critical Path Analysis (TimeQuest):
- Register File Read: 2.5ns
- ALU Operation: 8.2ns
- Memory Access: 5.1ns
- Register File Write: 2.1ns
Total: 17.9ns (55.8 MHz max frequency)
Target: 50 MHz ✓
```

### Verification Results
- ✅ All R-type instructions functional
- ✅ Memory operations working correctly
- ✅ Function calls with proper return
- ✅ Nested function calls successful
- ✅ Timing constraints met

## Skills Demonstrated
- MIPS architecture implementation
- Datapath design with MUXes
- Control unit design
- Assembly programming
- FPGA deployment
- Timing analysis and optimization
- Hardware debugging
- Function call conventions

## Lessons Learned
1. **Register $31 Handling**: Critical for return address storage
2. **PC Control**: Multiple sources require careful MUX design
3. **Timing**: Register file access can be critical path
4. **Testing**: Assembly programs essential for functional verification