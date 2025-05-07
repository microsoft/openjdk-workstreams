# Automatic Heap Sizing (AHS)

This repository tracks the design, architecture, discussions, and heuristics surrounding Automatic Heap Sizing (AHS) for G1GC and related collectors.

## Goals

- Act as the public reference point for AHS discussions.
- Archive important design notes (e.g., `SoftMaxHeapSize`, `GCTimeRatio`).
- Visualize dependencies, control flows, and heuristics.
- Serve as a collaboration hub across AHS contributors.

## Structure

- `design-notes/`: Detailed architectural notes and design documentation.
  - `state-of-ahs/`
    - [`State-of-AHS-for-G1.md`](design-notes/state-of-ahs/State-of-AHS-for-G1.md): Comprehensive overview of current state, Microsoft's positions, next steps, and detailed component‑level analysis.
    - [`02-control-loop-design.md`](design-notes/state-of-ahs/02-control-loop-design.md): Detailed design document for the AHS control loop implementation.
- `specs/`
  - [`AHS-Principles.md`](specs/AHS-Principles.md): High‑level design principles and signal hierarchy.
- `graphs/`: Visualizations of AHS architecture and dependencies.
  - [`G1-Control-Flow-2025.png`](graphs/G1-Control-Flow-2025.png): Explains how the control loop works (SoftMax → controller → resize).
  - [`G1-Shrink-Dependency-2025.png`](graphs/G1-Shrink-Dependency-2025.png): Detailed engineering view showing all patches and dependencies.
  - [`G1-AHS-Umbrella-Overview.png`](graphs/G1-AHS-Umbrella-Overview.png): High-level overview of components that roll up to the AHS umbrella.

---

Maintained by Monica Beckwith
