---

title: "Custom 8-Bit Multicycle Processor"

date: 2024-08-01T00:00:00Z

draft: false

author: "William Lazcano"

tags:

&nbsp; - SystemVerilog

&nbsp; - FPGA

&nbsp; - Processor Design

&nbsp; - Digital Systems

image: /images/projects/processor.jpg

description: "Complete custom processor design with FSM-based control unit and matrix computation capabilities"

toc: true

---



\## Project Overview



Designed and implemented a complete custom 8-bit multicycle processor from scratch as my final project for CPE 300L. The processor features a 16-instruction ISA, finite state machine-based control unit, and the ability to perform 2×2 matrix operations.



!\[Processor Datapath](/images/projects/processor.jpg)



\## Technical Specifications



\*\*Architecture:\*\*

\- \*\*Instruction Set:\*\* 16 custom instructions

\- \*\*Memory:\*\* 512-byte unified memory

\- \*\*Data Width:\*\* 8-bit operations

\- \*\*Execution:\*\* Multicycle (1-4 cycles)

\- \*\*Platform:\*\* Altera DE2-115 FPGA



\## What I Built



Created a fully functional processor capable of executing assembly programs for matrix mathematics using only integer arithmetic.



\## Technologies Used



\- SystemVerilog

\- ModelSim

\- Quartus II

\- Altera DE2-115 FPGA

