# Physical I/O Reference

This document describes what each PLC input in the station dispatch interlock represents in the real world — the actual sensor, switch, or pushbutton behind each address, and why it's wired the way it is. It's meant to accompany the ladder logic in `README.md` by explaining the hardware reasoning, not just the boolean behavior.

## %I0.0, %I0.1, %I0.2 — Restraint_Row1 / Row2 / Row3

Each of these represents a physical sensor built into one row's lap bar or over-the-shoulder harness — the mechanism that keeps a rider from being ejected during inversions, drops, or sudden stops. On a real coaster this is typically a limit switch or proximity sensor mounted at the restraint's locking pin: when the bar is pulled down and the mechanical latch engages, the switch closes and reports "locked" back to the PLC.

The PLC never senses the bar directly — only whether that switch says locked or not. This is also why each row gets its own independent input instead of one shared sensor for the whole train: if a single rider's bar fails to latch, the system needs to identify *which* row failed, not just that a failure occurred somewhere.

## %I0.3 — Ready_PB

The physical pushbutton on the operator's control panel. The ride operator presses this after walking the platform and visually confirming every restraint is down — it's the deliberate human "go" signal, kept separate from the automatic restraint sensors so that dispatch always requires an explicit human decision, not just sensors agreeing.

## %I0.4 — ESTOP

Represents a physical emergency stop button — typically a large red mushroom-head button, often present at multiple locations (operator panel, platform, sometimes trackside). These are wired normally-closed: pressing the button doesn't send a "stop" signal so much as it physically interrupts a circuit that's normally closed. This is the hardware reason behind the NC convention used in the ladder logic — a cut wire, a dead connection, or an actual button press all produce the identical result of an open circuit, which the PLC reads as unsafe. The system fails safe by default rather than needing a signal to tell it something is wrong.

## %I0.5 — Gate_Closed

A sensor on the station's entry/exit gate — commonly a magnetic reed switch or limit switch confirming the gate is latched shut. This exists so the train can't be dispatched while the gate is open and someone could still be standing in the loading area.

## %I0.6 — Fault_Reset_PB

A separate physical pushbutton, usually distinct in color or shape from the ready button (often labeled "Reset" or "Acknowledge"). The operator presses this only after confirming the fault condition is actually resolved — it exists specifically so a latched fault requires deliberate human acknowledgment rather than clearing itself.

## Why this matters

Each of these physical devices has a specific real-world failure mode that the ladder logic is built around — a bar that looks locked but isn't, a wire that gets cut, a gate someone forces open. Good PLC safety logic isn't just a set of boolean conditions; it's a direct translation of what can physically go wrong with each piece of hardware, and how the system finds out when it does.
