---

title: "ARM Processor with Extended Instruction Set"

date: 2024-07-01

draft: false

description: "Single-cycle ARM processor implementation with custom EOR and LDRB instructions"

image: "/images/projects/arm.jpg"

---



\## Project Overview



Implemented a complete single-cycle ARM processor in Verilog as part of CPE 300L Lab 10. Extended the base ARM architecture by adding two new instructions: EOR (Exclusive OR) and LDRB (Load Register Byte), requiring modifications to the ALU, decoder, and datapath.



!\[ARM Execution Cycles](/images/projects/arm.jpg)

\*ARM processor execution cycle table showing instruction flow\*



\## Architecture Components



\### Core Components



\*\*Datapath:\*\*

\- 16 General-purpose registers (R0-R15)

\- Program Counter (PC) with increment logic

\- Instruction Register (IR)

\- ALU with extended operations

\- Data Memory (DMEM)

\- Instruction Memory (IMEM)



\*\*Control Unit:\*\*

\- Main Decoder - Instruction type detection

\- ALU Decoder - Operation selection

\- Conditional Logic - Flag-based execution



\### Instruction Execution Cycles



The processor executes instructions in a single cycle with the following flow:



1\. \*\*Fetch:\*\* PC reads instruction from memory

2\. \*\*Decode:\*\* Control unit generates control signals

3\. \*\*Execute:\*\* ALU performs operation

4\. \*\*Memory:\*\* Load/Store operations access data memory

5\. \*\*Writeback:\*\* Results written to register file



\## Extended Instructions



\### EOR (Exclusive OR)



\*\*Opcode:\*\* `0001` (Function bits 4:1)



\*\*Implementation:\*\*

\- Extended ALU from 2-bit to 3-bit control

\- Added XOR logic: `Result = A ^ B`

\- Updated ALU decoder to recognize EOR instruction

\- Maintains flag updates (N, Z flags)



\*\*Verilog Code:\*\*

```verilog

// ALU Decoder - EOR Addition

case(Funct\[4:1]) 

&nbsp; 4'b0100: ALUControl\_reg = 3'b000; // ADD

&nbsp; 4'b0010: ALUControl\_reg = 3'b001; // SUB

&nbsp; 4'b0000: ALUControl\_reg = 3'b010; // AND

&nbsp; 4'b1100: ALUControl\_reg = 3'b011; // ORR

&nbsp; 4'b0001: ALUControl\_reg = 3'b100; // EOR (NEW)

&nbsp; default: ALUControl\_reg = 3'bx;

endcase

```



\### LDRB (Load Register Byte)



\*\*Purpose:\*\* Load a single byte from memory (vs full word)



\*\*Implementation Challenge:\*\*

\- ARM memory is word-aligned (32-bit)

\- Need to select correct byte based on address\[1:0]

\- Zero-extend byte to 32 bits



\*\*Byte Selection Logic:\*\*

```verilog

// LDRB byte selection based on address

case(ALUResult\[1:0])

&nbsp; 2'b00: ReadDataProcessed = {24'b0, ReadData\[7:0]};   // Byte 0

&nbsp; 2'b01: ReadDataProcessed = {24'b0, ReadData\[15:8]};  // Byte 1  

&nbsp; 2'b10: ReadDataProcessed = {24'b0, ReadData\[23:16]}; // Byte 2

&nbsp; 2'b11: ReadDataProcessed = {24'b0, ReadData\[31:24]}; // Byte 3

endcase

```



\## Test Program Results



\*\*Test Code:\*\*

```assembly

SUB R0, R15, R15      ; R0 = 0

ADD R2, R0, #5        ; R2 = 5

ADD R3, R0, #12       ; R3 = 12

EOR R6, R4, R2        ; R6 = 7 XOR 5 = 2

LDRB R8, \[R0, #100]   ; Load byte from memory

```



\*\*Execution Results:\*\*

\- All arithmetic operations verified correct

\- EOR produced expected XOR results

\- LDRB successfully loaded individual bytes

\- Flag updates functioned properly (Z, N flags)



\## Design Modifications



\### Main Decoder Updates



Extended decoder to recognize LDRB instruction:

\- Both LDR and LDRB use same control signals

\- Differentiation happens in datapath byte selection

\- Memory read occurs normally, byte extracted after



\### ALU Expansion



\*\*Changes:\*\*

\- ALUControl expanded from 2 bits to 3 bits

\- Added XOR operation case

\- Maintained existing ADD, SUB, AND, ORR operations

\- Preserved flag generation logic



\### Datapath Additions



Added byte selection multiplexer:

\- Detects LDRB instruction from opcode

\- Uses address low bits to select byte

\- Zero-extends result to 32 bits

\- Passes through full word for normal LDR



\## Challenges \& Solutions



\### Challenge 1: ALU Control Width

\*\*Problem:\*\* Adding EOR required 3-bit control, but design used 2-bit  

\*\*Solution:\*\* Updated all ALU control signals throughout hierarchy. Modified controller output, datapath connections, and ALU decoder.



\### Challenge 2: LDRB Byte Selection

\*\*Problem:\*\* How to detect LDRB vs LDR in datapath?  

\*\*Solution:\*\* Decode instruction opcode bits in datapath. Check for bit pattern: `Instr\[27:26] == 2'b01 \&\& Instr\[22] == 1'b1 \&\& Instr\[20] == 1'b1`



\### Challenge 3: Flag Updates

\*\*Problem:\*\* EOR should update N and Z flags but not C and V  

\*\*Solution:\*\* Modified flag write logic to only set N/Z for logical operations. C/V flags only update for arithmetic.



\## Verification \& Testing



\*\*ModelSim Simulation:\*\*

\- Created comprehensive testbench

\- Verified all 11 test instructions

\- Monitored internal signals (PC, ALUResult, flags)

\- Confirmed memory writes with expected values



\*\*FPGA Deployment:\*\*

\- Successfully synthesized on DE2-115 board

\- Executed test program on hardware

\- Observed results on 7-segment displays

\- Verified timing constraints met



\## What I Learned



\*\*ARM Architecture:\*\*

\- Deep understanding of RISC design principles

\- Load/Store architecture benefits

\- Conditional execution mechanism

\- Register file organization



\*\*Hardware Design:\*\*

\- Importance of hierarchical signal naming

\- Proper signal width management

\- Testbench development strategies

\- Timing constraint analysis



\*\*Debugging:\*\*

\- Waveform analysis techniques

\- Tracking signals through hierarchy

\- Identifying control signal conflicts

\- Systematic verification approach



\## Technologies Used



\- \*\*HDL:\*\* Verilog

\- \*\*Simulation:\*\* ModelSim (Altera Edition)

\- \*\*Synthesis:\*\* Quartus II

\- \*\*Platform:\*\* Altera DE2-115 FPGA

\- \*\*Testing:\*\* Custom testbench with waveform analysis



\## Code Repository



\[View Source Code on GitHub](https://github.com/Will-L10)



---



\*\*Project Duration:\*\* Summer 2024 (CPE 300L Lab 10)  

\*\*Achievement:\*\* 100% simulation accuracy, successful FPGA deployment

