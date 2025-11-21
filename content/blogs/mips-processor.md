---

title: "MIPS Processor with Extended ISA"

date: 2024-09-20

draft: false

author: "William Lazcano"

tags:

&nbsp; - MIPS

&nbsp; - FPGA

&nbsp; - SystemVerilog

&nbsp; - Pipelining

image: /images/projects/mips-processor.jpg

description: "5-stage pipelined MIPS processor with hazard detection and forwarding"

toc: true

---



\## Project Overview



Designed and implemented a 5-stage pipelined MIPS processor with hazard detection, data forwarding, and an extended instruction set on FPGA.



\## Architecture



\### Pipeline Stages

1\. \*\*Instruction Fetch (IF)\*\*: Retrieves instruction from memory

2\. \*\*Instruction Decode (ID)\*\*: Decodes instruction and reads registers

3\. \*\*Execute (EX)\*\*: Performs ALU operations

4\. \*\*Memory (MEM)\*\*: Accesses data memory

5\. \*\*Write Back (WB)\*\*: Writes results to register file



\### Hazard Handling

\- \*\*Data Hazards\*\*: Implemented forwarding paths to eliminate stalls

\- \*\*Control Hazards\*\*: Branch prediction with flush mechanism

\- \*\*Structural Hazards\*\*: Resolved through separate instruction/data memory



\## Implementation Details



\### Extended Instructions

Added custom instructions beyond standard MIPS ISA:

\- `MULT`: 32-bit multiplication

\- `DIV`: Integer division

\- `SWAP`: Register swap operation

\- Custom branch conditions



\### Hardware Specifications

\- \*\*Platform\*\*: Altera DE2-115 FPGA

\- \*\*Language\*\*: SystemVerilog

\- \*\*Clock Speed\*\*: 50 MHz

\- \*\*Register File\*\*: 32 x 32-bit registers



\## Code Snippet

```systemverilog

module mips\_pipeline (

&nbsp;   input logic clk,

&nbsp;   input logic reset,

&nbsp;   output logic \[31:0] pc\_out,

&nbsp;   output logic \[31:0] instr\_out

);

&nbsp;   // Pipeline registers

&nbsp;   logic \[31:0] if\_id\_instr, if\_id\_pc;

&nbsp;   logic \[31:0] id\_ex\_rs, id\_ex\_rt;

&nbsp;   logic \[31:0] ex\_mem\_alu\_result;

&nbsp;   logic \[31:0] mem\_wb\_read\_data;

&nbsp;   

&nbsp;   // Hazard detection unit

&nbsp;   hazard\_unit hu (

&nbsp;       .rs\_id(if\_id\_instr\[25:21]),

&nbsp;       .rt\_id(if\_id\_instr\[20:16]),

&nbsp;       .rt\_ex(id\_ex\_rt),

&nbsp;       .stall(stall),

&nbsp;       .flush(flush)

&nbsp;   );

&nbsp;   

&nbsp;   // Forwarding unit

&nbsp;   forwarding\_unit fu (

&nbsp;       .rs\_ex(id\_ex\_rs\_addr),

&nbsp;       .rt\_ex(id\_ex\_rt\_addr),

&nbsp;       .rd\_mem(ex\_mem\_rd),

&nbsp;       .rd\_wb(mem\_wb\_rd),

&nbsp;       .forward\_a(forward\_a),

&nbsp;       .forward\_b(forward\_b)

&nbsp;   );

endmodule

```



\## Performance Analysis



\### Metrics

\- \*\*CPI (Cycles Per Instruction)\*\*: ~1.2 (with forwarding)

\- \*\*Pipeline Efficiency\*\*: 83%

\- \*\*Branch Prediction Accuracy\*\*: 78%



\### Benchmarks

Tested with various programs:

\- Bubble sort algorithm

\- Matrix multiplication

\- Fibonacci sequence calculation



\## Testing Methodology



1\. \*\*Modular Testing\*\*: Each pipeline stage tested independently

2\. \*\*Integration Testing\*\*: Full pipeline validation

3\. \*\*Hazard Testing\*\*: Specific test cases for data/control hazards

4\. \*\*Performance Testing\*\*: Benchmark programs with timing analysis



\## Skills Demonstrated

\- Advanced digital design

\- Pipelining and hazard resolution

\- SystemVerilog programming

\- FPGA implementation

\- Performance optimization

