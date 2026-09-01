# plc-coaster-interlock
# Station Dispatch Interlock — PLC Simulation

A ladder logic simulation of a roller coaster station dispatch interlock, built to model the core safety-logic principles behind Independent Testable Restraint Systems (ITRS) used in amusement ride controls. Developed in Schneider Electric EcoStruxure Machine Expert, targeting a TM221CE16R PLC, and tested entirely in simulation mode (no physical hardware required).

## Overview

Before a coaster train can dispatch from the station, several independent conditions must all be true at once: every rider restraint is confirmed locked, the operator has pressed ready, the emergency stop circuit is intact, and the station gate is closed. If any safety condition is violated — most critically, an e-stop press — the system latches a fault that persists until an operator deliberately clears it, rather than silently resetting on its own. This project implements that logic from scratch in ladder diagram, and documents the reasoning behind each design choice.

## Hardware target

**TM221CE16R** — 9 digital inputs, 7 relay outputs (2A), 2 analog inputs, 100–240 VAC supply.

## I/O map

| Address | Tag | Type | Description |
|---|---|---|---|
| %I0.0 | Restraint_Row1 | Input | 1 = locked |
| %I0.1 | Restraint_Row2 | Input | 1 = locked |
| %I0.2 | Restraint_Row3 | Input | 1 = locked |
| %I0.3 | Ready_PB | Input | Operator dispatch pushbutton |
| %I0.4 | ESTOP | Input | Wired normally-closed; 1 = safe / not pressed |
| %I0.5 | Gate_Closed | Input | Station gate sensor |
| %I0.6 | Fault_Reset_PB | Input | Operator fault-reset pushbutton |
| %M0 | All_Restraints_OK | Memory | Internal flag, all restraints locked |
| %M1 | Fault_Latch | Memory | Latching fault flag |
| %Q0.0 | Dispatch_Permissive | Output (relay) | Allows launch/lift motor to engage |

## Design decisions worth noting

**E-stop wired normally-closed.** The `ESTOP` bit is defined so that `TRUE` means safe/not pressed. This mirrors real industrial practice: a broken wire or power loss reads as a stop condition rather than a false "all clear," so the system fails safe by default rather than failing open.

**Fault state uses SET/RESET, not a live coil.** A standard coil would drop the instant its rung condition goes false — meaning a momentary e-stop press would silently clear itself the moment the button was released, with no record anything happened. Using a SET coil to latch the fault, and a separate RESET rung to clear it, ensures a human has to deliberately acknowledge and clear the fault before dispatch can resume.

**Reset requires two conditions, not one.** The fault can only be cleared when the reset button is pressed *and* the e-stop is confirmed back in a safe state — preventing an operator from clearing a fault while the emergency condition is still active.

## Rung-by-rung logic

**Rung 0 — All Restraints OK**
Three restraint sensors in series (AND logic) drive an internal flag confirming every rider is locked in.

**Rung 1 — Dispatch Permissive**
Combines the restraint flag, ready pushbutton, e-stop status, and gate status (all AND), plus a normally-closed check that no fault is currently latched. Only when every condition holds does the dispatch-permissive output energize.

**Rung 2 — Set Fault Latch**
A normally-closed contact on the e-stop input triggers the moment the e-stop is pressed, latching the fault flag permanently until reset.

**Rung 3 — Reset Fault Latch**
Requires the reset pushbutton pressed *and* the e-stop confirmed safe, in series, before clearing the latched fault.

## Testing

All logic was verified in Machine Expert's simulation mode using forced I/O and a live watch table. Test sequence:

1. Force all three restraints TRUE → confirm `All_Restraints_OK` goes TRUE.
2. With restraints OK, force ready/e-stop-safe/gate-closed TRUE → confirm `Dispatch_Permissive` goes TRUE.
3. Drop any single condition → confirm `Dispatch_Permissive` drops immediately.
4. Force e-stop to unsafe, then back to safe → confirm `Fault_Latch` stays latched (does not self-clear).
5. Attempt reset while e-stop is still unsafe → confirm reset is rejected.
6. Restore e-stop to safe, then reset → confirm fault clears and dispatch can resume.

See `/screenshots` for ladder logic and watch-table captures from each stage of testing.

## Possible extensions

This project intentionally stays scoped to a single station's core interlock. Natural next steps include multi-train block-zone occupancy logic, lift hill anti-rollback monitoring, analog brake-pressure verification, and a supervisory HMI/SCADA layer — each of which would introduce additional ladder logic constructs (timers, counters, comparison blocks, function blocks) beyond the boolean interlock chain shown here.
