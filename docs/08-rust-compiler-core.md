# Rust compiler core

This page provides a comprehensive developer-oriented overview of the Rust compiler core within the `zhinst/laboneq` repository. It explains the architecture and purpose of the Rust crates, the PyO3 extension packaging, Cap'n Proto serialization for data exchange, the laboneq_rust submodules, the QCCS backend preprocessing, the scheduler and lowering passes, and the role of foreign/native components in the system. The goal is to clarify the Rust compiler core architecture, its design rationale, and its codebase organization, who consumes it, and what invariants it maintains.

---

## How to read this page as a maintainer

This page is structured to guide maintainers through the Rust compiler core's architecture and implementation details. It assumes familiarity with the overall LabOne Q architecture and Python DSL frontend, as covered in earlier pages. The content is organized by major components and their interactions, with references to source files and GitHub links for deeper inspection. Tables summarize crate responsibilities and interfaces, while Mermaid diagrams illustrate key data flows and module relationships.

Maintainers should use this page to understand the Rust compiler core's boundaries, its integration with Python via PyO3, and the data serialization mechanisms that enable efficient communication between Python and Rust. The explanations highlight design rationales and invariants to support safe and consistent evolution of the compiler core.

---

## Overview of the Rust compiler core

The Rust compiler core is the heart of the LabOne Q compilation pipeline, responsible for transforming the normalized Python DSL experiment representation into a scheduled, timed intermediate representation (IR), and ultimately producing device-specific code generation artifacts. It is implemented as a collection of Rust crates under `src/rust/` that collaborate to perform experiment lowering, scheduling, validation, and backend-specific preprocessing.

The Rust compiler core is exposed to Python as a single PyO3 extension module `laboneq._rust` with submodules for compiler and code generation functionality. This packaging strategy avoids multiple extension binaries and provides a seamless Python-Rust interoperability layer.

The core responsibilities include:

- Defining the Rust DSL experiment tree and IR node models.
- Performing backend-specific preprocessing, notably for the QCCS hardware backend.
- Scheduling experiments by resolving timing, parameter dependencies, and hardware constraints.
- Lowering the DSL tree into a timed IR suitable for code generation.
- Serializing and deserializing experiment data using Cap'n Proto schemas.
- Exposing compiler and scheduler APIs to Python via PyO3 bindings.

---

## Rust crates in the compiler core

The Rust compiler core is composed of several interrelated crates, each with a focused responsibility. Table 1 summarizes the primary crates involved in the compiler core.

| Crate Name                  | Location                                   | Purpose                                                                                         |
|-----------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------|
| `laboneq-ir`                | `src/rust/laboneq-ir/`                      | Defines the central intermediate representation (IR) types, timed tree nodes, and experiment container. |
| `laboneq-dsl`               | `src/rust/laboneq-dsl/`                     | Defines the Rust DSL experiment tree, operation variants, and frontend IR consumed by rewriting and scheduling. |
| `laboneq-scheduler`         | `src/rust/laboneq-scheduler/`               | Implements scheduling passes, validation, parameter resolution, loop unrolling, and lowering to IR nodes. |
| `laboneq-qccs-backend`      | `src/rust/laboneq-qccs-backend/`            | Provides QCCS hardware-specific backend preprocessing, device mapping, and signal metadata.      |
| `laboneq-compiler-py`       | `src/rust/laboneq-compiler-py/`             | PyO3 extension crate exposing compiler APIs to Python, including Cap'n Proto serialization and scheduling entry points. |
| `laboneq-rust`              | `src/rust/laboneq-rust/`                     | Root Rust crate aggregating core compiler functionality and PyO3 extension initialization.       |

Additional utility crates support logging, error handling, units, and Cap'n Proto schema bindings but are ancillary to the core compilation logic.

---

## PyO3 extension packaging and Python bridge

The Rust compiler core is exposed to Python as a single PyO3 extension module `laboneq._rust` with submodules `compiler` and `codegenerator`. This packaging is implemented in the `laboneq-compiler-py` crate (`src/rust/laboneq-compiler-py/src/lib.rs`) and the root crate `laboneq-rust` (`src/rust/laboneq-rust/src/lib.rs`).

The PyO3 bridge provides the following key functions:

- `build_experiment_capnp()`: Converts a serialized Cap'n Proto experiment payload from Python into Rust DSL and device setup structures, runs backend preprocessing, constructs the Rust experiment object, validates it, and returns a compiled experiment representation.
- `schedule_experiment()`: Takes a processed Rust experiment and near-time parameter dictionary, runs the scheduler, and returns a scheduled IR with timing and acquisition metadata.
- `serialize_experiment()`: Serializes Rust experiment structures back into Cap'n Proto bytes for Python consumption.
- `init_logging()`: Initializes Rust-side logging with configurable verbosity.

This design allows the Python compiler workflow to offload heavy scheduling and lowering logic to Rust while maintaining a Pythonic API surface.

---

## Cap'n Proto data exchange

Cap'n Proto is used as the serialization format for experiment data exchange between Python and Rust. The schemas are defined under `schemas/pulse/v1/` and compiled into Rust bindings in the `laboneq-capnp` crate (`src/rust/laboneq-capnp/`).

The Python side serializes the `ExperimentInfo` data structure (built by `ExperimentInfoBuilder` in Python) into Cap'n Proto bytes. The Rust compiler bridge deserializes these bytes into Rust DSL and device setup objects, enabling zero-copy or efficient data access.

Cap'n Proto's schema evolution and fast serialization support are critical for maintaining compatibility and performance in the compiler pipeline.

---

## laboneq_rust submodules and IR models

The Rust compiler core defines multiple IR layers and data models:

- **Rust DSL experiment tree** (`laboneq-dsl` crate): Represents the normalized experiment as a tree of `ExperimentNode` objects with `Operation` variants. Operations include structural nodes (e.g., `Root`, `Section`, `Sweep`, `Match`), pulse-level nodes (`PlayPulse`, `Acquire`, `Delay`), and near-time placeholders (`NearTimeCallback`). This tree is the frontend IR consumed by rewriting, validation, and scheduling.

- **Scheduled IR node tree** (`laboneq-ir` crate): After scheduling and lowering, the experiment is represented as a timed tree of `IrNode` objects with `IrKind` enums describing operations such as loops, pulse plays, acquisitions, and sections. Each node carries a `TinySamples` offset and length, encoding precise timing.

- **Experiment container** (`laboneq-ir` crate): The `ExperimentIr` struct bundles the scheduled root node with acquisition types, parameter stores, pulse definitions, and device setup context. It uses `Arc` for shared ownership to support Python bindings.

- **Pulse sheet schedule** (`laboneq-ir` crate): Optional data structure representing scheduled events for visualization in the pulse sheet viewer.

The IR models maintain invariants such as timing consistency, parameter resolution, and device setup correctness to ensure downstream code generation correctness.

---

## QCCS backend preprocessing

The QCCS backend preprocessing is implemented in the `laboneq-qccs-backend` crate (`src/rust/laboneq-qccs-backend/src/preprocessor.rs`). It performs hardware-specific mapping and validation before scheduling:

- Validates supported QCCS device combinations and rejects unsupported setups (e.g., ZQCS in QCCS backend).
- Handles special cases such as HDAWG+UHFQA without synchronization devices.
- Rejects incompatible signal combinations (e.g., HDAWG, UHFQA, and SHF signals together).
- Injects synthetic signals like `__small_system_trigger__` for desktop HDAWG/UHFQA setups.
- Creates compiler-visible AWG devices and assigns stable `AwgKey` identifiers.
- Parses port strings into channel lists and computes lead delays.
- Recognizes synchronization devices such as PQSC or QHUB auxiliary devices.

This preprocessing produces `QccsBackendPreprocessedData` containing backend signal metadata, device inventories, and timing offsets used by the scheduler and code generator.

---

## Scheduler and lowering passes

The scheduler and lowering passes are implemented primarily in the `laboneq-scheduler` crate (`src/rust/laboneq-scheduler/`).

### Scheduler responsibilities

The scheduler performs the following steps:

- Validates per-signal sampling-rate commensurability.
- Rejects strict timing sections outside the real-time boundary.
- Finds and clones the real-time subtree of the experiment.
- Resolves near-time match cases.
- Optionally chunks the experiment subtree for large sweeps.
- Determines acquisition types from averaging loops.
- Lowers the DSL tree into scheduled nodes (`ScheduledNode`).
- Applies repetition-mode resolution, IR validation, and acquisition-length adjustment.
- Performs loop unrolling and parameter resolution.
- Calculates timing and deduplicates warnings.
- Converts scheduled nodes into the final timed `IrNode` tree.

### Lowering semantics

Lowering transforms high-level DSL operations into timed IR nodes:

- Sections become `IrKind::Section` with triggers and schedule constraints.
- Loops (sweep, averaging, PRNG) become `IrKind::Loop` with iterations and preambles.
- Reserve nodes add reserved-signal constraints without IR operations.
- Play, acquire, and delay nodes become signal-level timed operations.
- `Delay` with `precompensation_clear=True` becomes a zero-duration `ClearPrecompensation` node followed by the delay.
- Reset oscillator phase and match nodes are lowered by specialized helpers.
- PRNG setup becomes a section with PRNG metadata.
- Loop preambles hold oscillator-frequency sweep setup, optional phase resets, and PPC sweep steps.
- Grid propagation escalates timing grids using least-common-multiple logic.

The lowering pass ensures that the final IR is a fully timed tree suitable for code generation.

---

## Foreign/native component role

The Rust compiler core acts as a foreign component to the Python DSL frontend and runtime controller. Its role is to provide a performant, safe, and precise compilation pipeline that Python orchestrates but does not implement directly.

The foreign/native boundary is managed via:

- Cap'n Proto serialization for experiment data exchange.
- PyO3 bindings exposing Rust APIs as Python submodules.
- Shared ownership of data structures using `Arc` to manage lifetimes across language boundaries.
- Python wrappers that invoke Rust compilation and scheduling functions, then repackage results into Python-native data structures for runtime consumption.

This design balances Python's flexibility and Rust's performance and safety, enabling complex scheduling and code generation while maintaining a Pythonic user experience.

---

## Source file and module locations

Table 2 lists key source files and modules relevant to the Rust compiler core.

| Component                         | Source Path                                                                                     | Description                                                                                   |
|----------------------------------|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| Rust compiler root crate          | [`src/rust/laboneq-rust/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs) | Aggregates core compiler functionality and PyO3 extension initialization.                     |
| PyO3 compiler bridge              | [`src/rust/laboneq-compiler-py/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs) | Exposes compiler APIs to Python, handles Cap'n Proto deserialization and scheduling calls.    |
| IR definitions and nodes         | [`src/rust/laboneq-ir/src/ir.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs), [`node.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs) | Defines `IrKind`, `IrNode`, and related IR data structures.                                  |
| Experiment container             | [`src/rust/laboneq-ir/src/experiment.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/experiment.rs) | Defines `ExperimentIr` bundling IR and context.                                              |
| Rust DSL experiment tree         | [`src/rust/laboneq-dsl/src/operation/variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs) | Defines Rust DSL operation variants and experiment node types.                               |
| Scheduler implementation         | [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs) | Implements scheduling passes and IR validation.                                              |
| Lowering passes                  | [`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs) | Implements lowering from DSL tree to timed IR nodes.                                        |
| QCCS backend preprocessing      | [`src/rust/laboneq-qccs-backend/src/preprocessor.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs) | Performs hardware-specific backend preprocessing and device mapping.                         |

---

## Data flow and module interaction diagram

The following Mermaid diagram illustrates the data flow and module interactions in the Rust compiler core, highlighting the Python-Rust boundary and major processing stages.

```mermaid
flowchart TD
    PY[Python DSL ExperimentInfo]
    CP[Cap'n Proto Serialization]
    RB[laboneq-compiler-py PyO3 Bridge]
    DSL[laboneq-dsl Rust DSL Tree]
    QCCS[QCCS Backend Preprocessor]
    SCH[laboneq-scheduler Scheduler & Lowering]
    IR[laboneq-ir Timed IR & Experiment Container]
    CG[Code Generator (Rust & Python wrapper)]
    PYRT[Python Runtime Controller]

    PY -->|Serialize| CP
    CP -->|Deserialize| RB
    RB --> DSL
    DSL --> QCCS
    QCCS --> SCH
    SCH --> IR
    IR --> CG
    CG --> PYRT
```

---

## Detailed component descriptions

### laboneq-ir crate: Intermediate representation

The `laboneq-ir` crate defines the core IR types representing the scheduled experiment as a timed tree. The central type is `IrNode`, which stores an `IrKind` enum variant, a `TinySamples` length, and a vector of `NodeChild` records. Each child is attached at a `TinySamples` offset relative to its parent, forming a timed tree structure.

The `IrKind` enum covers a wide range of operations, including:

- Structural nodes: `Root`, `Section`, `Loop`, `Match`, `Case`.
- Signal-level operations: `PlayPulse`, `Acquire`, `Delay`.
- Setup and control operations: `SetOscillatorFrequency`, `ResetOscillatorPhase`, `PpcStep`.
- PRNG setup and clearing precompensation.

The `ExperimentIr` struct bundles the root `IrNode` with acquisition types, parameter stores, pulse definitions, and device setup context. Shared ownership via `Arc` supports Python bindings and cross-thread usage.

This crate is the canonical representation of the scheduled experiment passed to code generation.

### laboneq-dsl crate: Rust DSL experiment tree

The `laboneq-dsl` crate defines the Rust DSL experiment tree consumed by rewriting, validation, and scheduling. It models the experiment as a tree of `ExperimentNode` objects with `Operation` variants.

Operations include:

- Structural: `Root`, `Section`, `PrngSetup`, `PrngLoop`, `Sweep`, `AveragingLoop`, `RealTimeBoundary`, `Match`, `Case`.
- Pulse-level: `Reserve`, `PlayPulse`, `Acquire`, `Delay`, `ResetOscillatorPhase`.
- Near-time placeholders: `NearTimeCallback`, `SetNode`.

Each operation carries semantic data such as alignment, triggers, sweep parameters, repetition modes, oscillator phase controls, and acquisition handles.

Helper methods expose signal usage, section identity, loop identity, and real-time compatibility validation.

### laboneq-qccs-backend crate: Hardware-specific preprocessing

The QCCS backend preprocessing is critical for mapping the generic experiment representation to the specific hardware constraints of Zurich Instruments QCCS devices.

Key features include:

- Validation of device combinations and rejection of unsupported setups.
- Injection of synthetic signals for synchronization.
- Construction of `AwgDevice` objects representing compiler-visible AWG devices.
- Assignment of stable `AwgKey` identifiers based on channel and device UID grouping.
- Computation of lead delays and channel parsing.
- Recognition of synchronization devices such as PQSC and QHUB.

This preprocessing produces `QccsBackendPreprocessedData` used downstream by the scheduler and code generator.

### laboneq-scheduler crate: Scheduling and lowering

The scheduler orchestrates the transformation from the Rust DSL experiment tree to the timed IR.

Its responsibilities include:

- Validating sampling rates and timing constraints.
- Extracting the real-time subtree.
- Resolving near-time match cases.
- Chunking large sweeps.
- Determining acquisition types.
- Lowering DSL nodes to `ScheduledNode` objects.
- Applying repetition mode resolution and IR validation.
- Adjusting acquisition lengths.
- Unrolling loops.
- Resolving parameters.
- Calculating timing offsets.
- Deduplicating warnings.
- Converting scheduled nodes to `IrNode` trees.

The lowering pass injects initial oscillator and voltage offset setup nodes and transforms high-level DSL operations into timed IR nodes with precise offsets.

### laboneq-compiler-py crate: Python-Rust bridge

This crate exposes Rust compiler functionality to Python via PyO3. It provides:

- Functions to deserialize Cap'n Proto experiment data into Rust DSL and device setup objects.
- Backend-specific preprocessing hooks.
- Experiment construction and validation.
- Scheduling entry points accepting near-time parameters and chunking info.
- Serialization of Rust experiment data back to Cap'n Proto.
- Logging initialization.

The PyO3 bridge is the gateway for Python to invoke Rust compilation and scheduling logic efficiently.

---

## Invariants and design considerations

The Rust compiler core maintains several important invariants:

- **Timing consistency**: All IR nodes have precise `TinySamples` offsets and lengths relative to their parents, ensuring sample-accurate scheduling.
- **Parameter resolution**: Parameters are resolved and propagated consistently through scheduling and lowering passes.
- **Device setup correctness**: The device setup context is validated and stable, with consistent signal and device identifiers.
- **Backend compatibility**: The QCCS backend preprocessing enforces hardware constraints and rejects unsupported configurations.
- **Safe ownership**: Shared data structures use `Arc` to manage lifetimes across Python and Rust boundaries.
- **Separation of concerns**: The Rust compiler core focuses on compilation and scheduling, leaving runtime execution and device communication to Python controller components.

---

## Summary table: Rust compiler core components

| Component                  | Role                                    | Location                                      | Consumers                   |
|----------------------------|-----------------------------------------|-----------------------------------------------|-----------------------------|
| `laboneq-ir`               | Timed IR node tree and experiment container | `src/rust/laboneq-ir/`                         | Scheduler, code generator   |
| `laboneq-dsl`              | Rust DSL experiment tree and operations | `src/rust/laboneq-dsl/`                        | Scheduler                   |
| `laboneq-qccs-backend`     | QCCS hardware backend preprocessing     | `src/rust/laboneq-qccs-backend/`              | Scheduler                   |
| `laboneq-scheduler`        | Scheduling passes, lowering, validation | `src/rust/laboneq-scheduler/`                  | PyO3 bridge, code generator |
| `laboneq-compiler-py`      | PyO3 bridge exposing Rust compiler APIs | `src/rust/laboneq-compiler-py/`                | Python compiler workflow    |
| `laboneq-rust`             | Root Rust crate and PyO3 extension init | `src/rust/laboneq-rust/`                        | PyO3 bridge                 |

---

## Practical developer orientation

### What exists?

- A modular Rust compiler core composed of crates for IR, DSL, scheduling, backend preprocessing, and Python bridging.
- A PyO3 extension exposing compiler and scheduler APIs to Python.
- Cap'n Proto schemas and serialization for efficient data exchange.
- Backend-specific preprocessing for QCCS hardware.
- Timed IR node trees representing scheduled experiments.
- Lowering passes transforming DSL trees into timed IR.

### Why does it exist?

- To provide a performant, safe, and precise compilation pipeline for LabOne Q experiments.
- To offload complex scheduling and timing logic from Python to Rust.
- To enable sample-accurate pulse scheduling and hardware-specific code generation.
- To maintain a clean separation between experiment definition (Python DSL) and compilation/execution.
- To support hardware-specific constraints and optimizations via backend preprocessing.

### Where does it live?

- Rust crates under `src/rust/` in the repository root.
- PyO3 bridge in `src/rust/laboneq-compiler-py/`.
- IR and DSL crates in `src/rust/laboneq-ir/` and `src/rust/laboneq-dsl/`.
- Scheduler in `src/rust/laboneq-scheduler/`.
- Backend preprocessing in `src/rust/laboneq-qccs-backend/`.
- Root crate and extension initialization in `src/rust/laboneq-rust/`.

### Who consumes it?

- The Python compiler workflow (`src/python/laboneq/compiler/workflow/`) invokes Rust APIs via PyO3.
- The Python scheduler wrapper calls Rust scheduling functions.
- The Python code generator wrapper consumes the scheduled IR for device-specific code generation.
- Runtime controller components consume compiled experiment artifacts produced downstream.
- Developers extending the compiler or adding backend support interact with Rust crates.

### What invariants does it carry?

- Precise timing and offset representation in IR nodes.
- Consistent parameter resolution and propagation.
- Stable device and signal identifiers.
- Backend-specific hardware constraints enforced.
- Safe memory ownership across Python-Rust boundary.
- Separation of scheduling and code generation concerns.

---

## References used on this page

1. LabOne Q repository, Rust compiler root crate: [`src/rust/laboneq-rust/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs)
2. PyO3 compiler bridge: [`src/rust/laboneq-compiler-py/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs)
3. Rust IR crate: [`src/rust/laboneq-ir/`](https://github.com/zhinst/laboneq/tree/main/src/rust/laboneq-ir)
4. Rust DSL operation variants: [`src/rust/laboneq-dsl/src/operation/variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs)
5. Scheduler implementation: [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs)
6. Lowering passes: [`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs)
7. QCCS backend preprocessing: [`src/rust/laboneq-qccs-backend/src/preprocessor.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs)
8. Python compiler workflow: [`src/python/laboneq/compiler/workflow/compiler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py)
9. Python scheduler wrapper: [`src/python/laboneq/compiler/scheduler/scheduler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/scheduler/scheduler.py)
10. Python code generator wrapper: [`src/python/laboneq/compiler/seqc/code_generator.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/seqc/code_generator.py)

---

This concludes the detailed overview of the Rust compiler core in LabOne Q. For further details, maintainers are encouraged to explore the referenced source files and crates.
