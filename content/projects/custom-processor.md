---

title: "Custom 8-Bit Multicycle Processor"

date: 2024-08-01

draft: false

description: "Complete custom processor design with FSM-based control unit and matrix computation capabilities"

tags: \["SystemVerilog", "FPGA", "Processor Design", "Digital Systems"]

---



\## Project Overview



Designed and implemented a complete custom 8-bit multicycle processor from scratch with a 16-instruction ISA, FSM-based control unit, and the ability to perform 2×2 matrix operations (addition, subtraction, multiplication, transposition, and inversion).



\## Key Features



\- \*\*Custom Instruction Set:\*\* 16 instructions including ADD, SUB, MULT, DIV, AND, OR, XOR, NOT, JUMP, JMPZ, JPNZ, LDAC, STAC, MVAC, MOVR, INAC, CLAC

\- \*\*Multicycle Execution:\*\* Variable cycle count (1-4 cycles) based on instruction complexity

\- \*\*512-Byte Unified Memory:\*\* Byte-addressable shared instruction and data storage

\- \*\*Matrix Computations:\*\* Software implementation of 2×2 matrix operations using integer arithmetic

\- \*\*FPGA Deployment:\*\* Successfully deployed on Altera DE2-115 with programmable memory interface



\## Technical Implementation



\### Architecture Components



1\. \*\*Control Unit:\*\* FSM with 6 states orchestrating datapath operations

2\. \*\*Datapath:\*\* 11 main components including PC, IR, ALU, registers, and multiplexers

3\. \*\*Memory System:\*\* 512-byte unified memory with programming mode support

4\. \*\*ALU:\*\* 8-bit arithmetic and logic operations with zero flag



\### Design Highlights



\- Implemented complete FSM for instruction fetch, decode, and execution

\- Created custom datapath with flexible mux-based data routing

\- Developed assembly programs for matrix operations entirely in software

\- Added hardware debugging features with 7-segment display output

\- Achieved stable operation with manual clock control via FPGA switches



\## Results



Successfully executed matrix computation programs on physical FPGA hardware, demonstrating:

\- Correct arithmetic operations across all instruction types

\- Proper conditional branching and control flow

\- Accurate memory read/write operations

\- Real-time result display on 7-segment displays



\## Code



\[View on GitHub](https://github.com/Will-L10)



\## Gallery



!\[Custom Processor Datapath](/images/projects/processor.jpg)

\*Complete datapath architecture showing all components and signal routing\*



\## What I Learned



\- Deep understanding of processor architecture from ground up

\- FSM design for complex control logic

\- Hardware-software co-design principles

\- FPGA debugging and verification techniques

\- Trade-offs between hardware complexity and software flexibility

