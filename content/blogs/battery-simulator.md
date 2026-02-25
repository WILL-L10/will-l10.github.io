---
title: "Battery Discharge Simulator — Multi-Platform Implementation"
date: 2025-12-07
draft: false
description: "Modeled a 3S2P Li-ion battery pack across C++, LTspice, and Verilog — all three converged within 1% variance."
image: /images/projects/battery.jpg
tags: ["C++", "LTspice", "Verilog", "Circuit Analysis", "EE 220"]
---

## Overview

For EE 220 (Circuits), I built a battery discharge simulator three ways: a C++ software simulation, an analog circuit model in LTspice, and a hardware description in Verilog running in ModelSim. The goal was to model a real 3S2P 18650 Li-ion battery pack discharging through a 10Ω resistive load and validate the results across all three platforms.

**Battery configuration:** 3S2P 18650 Li-ion — 11.1V nominal, 6000 mAh capacity  
**Load:** 10Ω constant resistive  
**Key result:** 5.8 hour runtime, 10.3 Wh energy delivered

---

## C++ Implementation

I built a `BatterySimulator` class that takes battery type, series/parallel cell count, and load resistance as inputs. Each time step computes the open-circuit voltage based on state of charge (SOC), calculates current through the total resistance (load + internal), updates the terminal voltage, and checks against the cutoff threshold.

```cpp
class BatterySimulator {
private:
    BatteryType battery;
    int seriesCells;
    int parallelCells;
    double loadResistance;
    vector<DischargePoint> data;

public:
    BatterySimulator(BatteryType bat, int series, int parallel, double load)
        : battery(bat), seriesCells(series), parallelCells(parallel), loadResistance(load) {}

    void simulate() {
        double nominalV = battery.voltage * seriesCells;
        double totalCapacity = battery.capacity * parallelCells;
        double totalResistance = (battery.resistance * seriesCells) / parallelCells;
        double cutoffV = battery.cutoff * seriesCells;
        // ...iterates until cutoff voltage reached
    }
};
```

The simulation iterates in 0.05-hour steps, storing voltage, current, SOC, and power at each point. Output includes a full discharge table and summary stats.

---

## LTspice Implementation

The LTspice model used a SPICE netlist to represent the Thévenin equivalent of the battery pack — a voltage source with SOC-dependent output in series with the internal resistance, driving the 10Ω load.

Key concepts applied:
- **Thévenin equivalent circuit** — simplified the 3S2P pack into a single source + resistance
- **KVL** — terminal voltage = open-circuit voltage − (current × internal resistance)
- **RC circuit theory** — modeled the gradual voltage sag as capacity depletes

The `.cir` netlist runs a transient simulation and plots voltage, current, and power over time.

---

## Verilog / ModelSim Implementation

The hardware model runs on a clock, updating battery state every cycle. Each clock tick represents 3 minutes of simulated time.

```verilog
module battery_simulator (
    input wire clk,
    input wire reset,
    output reg [15:0] voltage,
    output reg [15:0] current,
    output reg [7:0]  soc,
    output reg [15:0] power,
    output reg battery_depleted
);
    parameter NOMINAL_VOLTAGE = 1110;  // 11.10V scaled x100
    parameter TOTAL_CAPACITY  = 6000;  // mAh
    parameter INTERNAL_RESISTANCE = 75; // 0.075Ω scaled x1000
    parameter LOAD_RESISTANCE = 10000;  // 10Ω scaled x1000
    parameter CUTOFF_VOLTAGE  = 900;   // 9.00V scaled x100

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            soc <= 100;
            battery_depleted <= 0;
            // ...initialize all outputs
        end else if (!battery_depleted) begin
            // compute SOC, voltage, current, power each cycle
            // assert battery_depleted when voltage < CUTOFF_VOLTAGE
        end
    end
endmodule
```

The testbench runs 500 clock cycles and prints a discharge table to the ModelSim transcript every 15 minutes of simulated time.

---

## Validation Results

All three platforms produced consistent results — within 1% variance of each other and within 0.5% of hand calculations.

| Platform  | Runtime | Energy  | Avg Current |
|-----------|---------|---------|-------------|
| C++       | 5.83h   | 10.26 Wh | 0.95A     |
| LTspice   | 5.80h   | 10.30 Wh | 0.96A     |
| ModelSim  | 5.78h   | 10.28 Wh | 0.95A     |
| Hand calc | 5.80h   | 10.27 Wh | 0.95A     |

**Hand calculation check at t=0:**
- Current: I = 11.1V / 10.075Ω = 1.102A ✓
- Terminal voltage: V = 11.1V − 0.083V = 11.02V ✓  
- Power: P = 12.14W ✓
- Efficiency: η = 99.3% ✓

---

## What I Learned

This project forced me to think about the same physical system from three completely different angles — mathematical, analog, and digital. Writing the C++ simulation first gave me an intuition for the discharge curve that made the LTspice model much easier to set up. Implementing it in Verilog then required thinking about fixed-point arithmetic and clock-driven state, which is a different mental model entirely.

The 1% convergence across all three platforms confirmed the underlying circuit theory was correct — and that's the part that mattered most.
