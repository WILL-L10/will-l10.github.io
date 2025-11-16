---

title: "MIPS Processor with Subroutine Support"

date: 2024-06-01

draft: false

description: "Complete single-cycle MIPS CPU with JAL/JR instructions for function calls"

image: "/images/projects/mips.jpg"

---



\## Project Overview



Built a complete single-cycle MIPS processor from scratch for CPE 300L Lab 7. Extended the base design by implementing JAL (Jump and Link) and JR (Jump Register) instructions, enabling subroutine calls and returns - essential for structured programming.



!\[MIPS Processor](/images/projects/mips.jpg)

\*MIPS processor running on DE2-115 FPGA with 7-segment display output\*



\## MIPS Architecture Overview



\*\*Design Philosophy:\*\*

\- RISC (Reduced Instruction Set Computer)

\- Load/Store architecture

\- 32 general-purpose registers

\- Harvard architecture (separate instruction and data memory)

\- Fixed 32-bit instruction format



\*\*Implemented Instruction Set:\*\*

\- \*\*R-Type:\*\* ADD, SUB, AND, OR, SLT

\- \*\*I-Type:\*\* ADDI, LW, SW, BEQ

\- \*\*J-Type:\*\* J, JAL (added), JR (added)



\## Subroutine Implementation



\### JAL (Jump and Link)



\*\*Purpose:\*\* Call a subroutine and save return address



\*\*Operation:\*\*

```

$31 (ra) = PC + 4  # Save return address

PC = jump\_target    # Jump to subroutine

```



\*\*Hardware Requirements:\*\*

\- Write PC+4 to register $31

\- Load jump target into PC

\- Requires modifications to register file write logic



\### JR (Jump Register)



\*\*Purpose:\*\* Return from subroutine



\*\*Operation:\*\*

```

PC = $rs  # Jump to address in register (typically $31)

```



\*\*Hardware Requirements:\*\*

\- Read register value

\- Load into PC

\- Bypass normal PC increment



\## Implementation Details



\### Control Unit Modifications



\*\*New Control Signals:\*\*

\- `RegDst` - Select destination register (rd vs rt vs $31)

\- `Jump` - Enable jump operation

\- `JumpReg` - Select register-based jump

\- `Link` - Enable writing return address



\*\*JAL Control Logic:\*\*

```verilog

// JAL writes PC+4 to $31

assign RegDst = (opcode == JAL) ? 2'b10 : normal\_logic;

assign MemtoReg = (opcode == JAL) ? 1'b0 : normal\_logic;

assign RegWrite = (opcode == JAL) ? 1'b1 : normal\_logic;

```



\### Datapath Extensions



\*\*Added Components:\*\*

1\. \*\*MUX for Register Destination:\*\*

&nbsp;  - Normal: rd or rt

&nbsp;  - JAL: hardwired to register 31



2\. \*\*MUX for Write Data:\*\*

&nbsp;  - Normal: ALU result or memory data

&nbsp;  - JAL: PC + 4



3\. \*\*MUX for PC Source:\*\*

&nbsp;  - Normal increment

&nbsp;  - Branch target

&nbsp;  - Jump target

&nbsp;  - Register value (JR)



\## Test Program: Function Call Demo



\*\*Objective:\*\* Compute (a + b) \* 2 using subroutine



\*\*Assembly Code:\*\*

```assembly

main:

&nbsp;   addi $4, $0, 15         # a = 15

&nbsp;   addi $5, $0, 25         # b = 25

&nbsp;   jal add\_and\_double      # Call function

&nbsp;   sw $2, 80($0)           # Store result

&nbsp;   j end\_program



add\_and\_double:

&nbsp;   add $2, $4, $5          # sum = a + b = 40

&nbsp;   add $2, $2, $2          # result = sum \* 2 = 80

&nbsp;   jr $31                  # Return



end\_program:

&nbsp;   j end\_program           # Infinite loop

```



\*\*Expected Execution:\*\*

1\. PC=0: Set a=15

2\. PC=4: Set b=25

3\. PC=8: JAL saves PC+4 (=12) in $31, jumps to 24

4\. PC=24: Add 15+25=40

5\. PC=28: Add 40+40=80

6\. PC=32: JR jumps to address in $31 (=12)

7\. PC=12: Store 80 to memory\[80]

8\. PC=16: Store 80 to memory\[84]

9\. PC=20: Jump to end



\*\*Results:\*\* ✅ All steps verified in simulation and hardware



\## Hardware Deployment



\### DE2-115 FPGA Implementation



\*\*Display Mapping:\*\*

\- SW\[17:16] select display mode:

&nbsp; - 00: Program Counter

&nbsp; - 01: Current Instruction

&nbsp; - 10: Write Data

&nbsp; - 11: Data Address



\*\*7-Segment Display:\*\*

\- HEX7-HEX0: Show selected value in hexadecimal

\- LEDG\[8]: Flashes on memory write

\- RED LEDs: Show switch states



\### Timing Optimization



\*\*TimeQuest Analysis:\*\*

\- Identified critical path: Register File → ALU → Memory

\- Maximum frequency: ~50 MHz

\- Optimized by adding pipeline registers (future enhancement)



\*\*Clock Division:\*\*

\- Divided 50MHz board clock to 1Hz for observation

\- Implemented pause-on-write for debugging

\- Resume button for step-by-step execution



\## Challenges \& Solutions



\### Challenge 1: Return Address Management

\*\*Problem:\*\* How to save PC+4 and write to $31 simultaneously?  

\*\*Solution:\*\* Added MUX to select between rd, rt, and hardcoded 31. Added MUX to select between ALU result and PC+4 for register write data.



\### Challenge 2: PC Update Priority

\*\*Problem:\*\* Multiple sources want to update PC (increment, branch, jump, JR)  

\*\*Solution:\*\* Created priority encoder: JR > JAL > Branch > Increment. Used nested MUXes with proper control signal precedence.



\### Challenge 3: Stuck Processor State

\*\*Problem:\*\* Processor would hang on certain instruction sequences  

\*\*Solution:\*\* Added watchdog timer and stuck detection. Implemented manual clock control for debugging. Found issue was incorrect JR control signal timing.



\## Performance Metrics



\*\*Resource Utilization (Cyclone IV):\*\*

\- Logic Elements: 2,847 / 114,480 (2%)

\- Registers: 1,023

\- Memory Bits: 16,384 (for instruction/data memory)

\- Maximum Clock: 52.3 MHz



\*\*Instruction Execution:\*\*

\- Cycles per instruction: 1 (single-cycle design)

\- Memory access time: 10ns

\- Total instruction latency: ~19ns



\## What I Learned



\*\*Processor Design:\*\*

\- Single-cycle vs multi-cycle trade-offs

\- Control signal generation and timing

\- Datapath optimization techniques

\- Memory hierarchy considerations



\*\*MIPS Architecture:\*\*

\- Instruction encoding formats

\- Register conventions ($ra, $sp, $gp)

\- Calling conventions and stack frames

\- Assembly programming techniques



\*\*Debugging Strategies:\*\*

\- Waveform-based debugging

\- Hierarchical signal tracing

\- Incremental testing approach

\- Hardware-software co-verification



\*\*System Integration:\*\*

\- FPGA resource management

\- I/O interfacing (switches, LEDs, displays)

\- Clock domain crossing

\- Real-time debugging techniques



\## Technologies Used



\- \*\*HDL:\*\* Verilog

\- \*\*Simulation:\*\* ModelSim

\- \*\*Synthesis:\*\* Quartus II  

\- \*\*Timing Analysis:\*\* TimeQuest Analyzer

\- \*\*Platform:\*\* Altera DE2-115 FPGA (Cyclone IV E)



\## Future Enhancements



\- Pipeline implementation (5-stage)

\- Hazard detection and forwarding

\- Branch prediction

\- Cache memory hierarchy

\- Interrupt handling



\## Code Repository



\[View Source Code on GitHub](https://github.com/Will-L10)



---



\*\*Project Duration:\*\* Summer 2024 (CPE 300L Lab 7)  

\*\*Achievement:\*\* Full functionality demonstrated on hardware

