# Intermediate representations

This chapter provides a comprehensive taxonomy and semantic overview of the intermediate representations (IRs) used throughout the `zhinst/laboneq` codebase. These IRs form the backbone of the LabOne Q compiler and runtime system, bridging the user-facing Python domain-specific language (DSL) and the low-level hardware execution. Understanding these IRs is crucial for developers working on the compiler, scheduler, code generation, and runtime execution layers.

We cover the following key IRs and data structures:

- Python DSL experiment tree
- `ExperimentInfo` payload
- Rust DSL operation tree
- Scheduled IR node tree (`IrNode`)
- Schedule map and recipe
- Code generation artifacts
- Near-time execution IR
- Runtime `RecipeData` and `ScheduledExperiment`

This chapter is the most important in the guide, as it explains the layered abstractions and transformations that enable LabOne Q to compile and execute complex quantum experiments on Zurich Instruments hardware.

---

## How to read this page as a maintainer

This page is structured to first introduce the high-level Python DSL IR that users and experiment authors interact with, then progressively describe the internal compiler IRs in Rust, and finally the runtime data structures used for execution and control. Each section explains:

- **What** the IR or data structure represents
- **Why** it exists and its role in the compilation/execution pipeline
- **Where** it is implemented in the codebase (with file paths and source links)
- **Who** consumes it (which components or layers)
- **What invariants** or semantic guarantees it carries

Mermaid diagrams illustrate the relationships and data flow between IRs. Inline code and source links provide concrete entry points for exploration.

Maintainers should use this chapter as a reference when modifying or extending the compiler or runtime, ensuring that new features respect the IR semantics and integration points.

---

## 1. Python DSL experiment tree

### What exists

The Python DSL experiment tree is the user-facing representation of an experiment. It is constructed by the user via the `laboneq.dsl.experiment.Experiment` class and its contained `Section` objects and operations.

- The root container is [`Experiment`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py).
- Sections are instances of [`Section`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py) or its subclasses.
- Operations such as `play()`, `acquire()`, `delay()`, `reset_oscillator_phase_()`, `call()`, and `set_node()` are appended as children to sections.

The DSL supports constructs for:

- Real-time loops (`AcquireLoopRt`)
- Near-time parameter sweeps (`Sweep`)
- Conditional execution (`Match`, `Case`)
- Pseudo-random number generation (`PRNGSetup`, `PRNGLoop`)

### Why it exists

This DSL tree provides a high-level, declarative interface for experiment authors to describe pulse sequences, measurement loops, and control flow without hardware-specific details. It abstracts away device setup and scheduling concerns.

### Where it lives

- Python package: `src/python/laboneq/dsl/experiment/`
- Key files:
  - [`experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py)
  - [`section.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py)

### Who consumes it

- The `ExperimentInfoBuilder` in the payload builder layer consumes the DSL tree to produce the next IR.
- The Python compiler workflow (`src/python/laboneq/compiler/workflow/compiler.py`) initiates compilation from this DSL.

### Invariants and semantics

- The tree is a nested structure of sections and operations.
- Real-time loops are restricted to a single `AcquireLoopRt` per experiment.
- Near-time sections cannot be nested under real-time loops.
- Section timing modes and alignment are specified but timing constraints are enforced later by the scheduler.

---

## 2. ExperimentInfo payload

### What exists

`ExperimentInfo` is a Python data structure representing a normalized and validated payload derived from the DSL tree and device setup. It contains:

- Logical and physical signal mappings
- Calibration data merged from baseline and experiment-specific calibrations
- Parameter information for sweeps and driven parameters
- Chunking and acquire-loop metadata
- Device setup fingerprints

It is produced by the [`ExperimentInfoBuilder`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py).

### Why it exists

`ExperimentInfo` serves as the compatibility bridge payload between the Python DSL and the Rust compiler backend. It consolidates all experiment and setup information into a form suitable for serialization and consumption by the Rust compiler.

### Where it lives

- Python package: `src/python/laboneq/implementation/payload_builder/experiment_info_builder/`
- Key file: [`experiment_info_builder.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py)

### Who consumes it

- The Python compiler workflow calls the compatibility bridge (`compat.py`) to convert `ExperimentInfo` into Rust DSL structures.
- The Rust compiler backend consumes the serialized Cap'n Proto representation of `ExperimentInfo`.

### Invariants and semantics

- Signals are uniquely mapped to physical channels; conflicts are detected.
- Calibration data is merged with priority rules.
- Parameter dependencies and chunking are fully resolved.
- The payload is immutable once built.

---

## 3. Rust DSL operation tree

### What exists

The Rust DSL operation tree is the normalized experiment representation inside the Rust compiler before scheduling. It consists of `ExperimentNode` objects, each with an `Operation` kind.

- Structural nodes: `Root`, `Section`, `PrngSetup`, `PrngLoop`, `Sweep`, `AveragingLoop`, `RealTimeBoundary`, `Match`, `Case`
- Pulse-level nodes: `Reserve`, `PlayPulse`, `Acquire`, `Delay`, `ResetOscillatorPhase`
- Near-time-only placeholders: `NearTimeCallback`, `SetNode`

Each operation carries semantic fields relevant to its type, such as section alignment, sweep parameters, acquisition type, repetition mode, pulse parameters, and more.

### Why it exists

This IR provides a normalized, strongly typed, and backend-optimized representation of the experiment suitable for rewriting, validation, and scheduling passes in Rust.

### Where it lives

- Rust crate: `laboneq-dsl`
- Source files:
  - [`src/rust/laboneq-dsl/src/operation/variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs)

### Who consumes it

- The Rust scheduler and lowering passes consume this DSL tree.
- Backend-specific preprocessing and validation operate on this IR.

### Invariants and semantics

- The tree is normalized with explicit operation kinds.
- Real-time compatibility is enforced.
- Loop and section identities are preserved.
- Timing constraints are not yet resolved.

---

## 4. Scheduled IR node tree (`IrNode`)

### What exists

The scheduled IR node tree is the core timed tree representation after scheduling and lowering. It consists of `IrNode` objects with:

- An `IrKind` enum specifying the operation kind (e.g., `Root`, `Section`, `Loop`, `PlayPulse`, `Acquire`, `Delay`, `Match`, `Case`, `ClearPrecompensation`)
- A `TinySamples` duration length
- A vector of `NodeChild` records, each with a relative offset and child node

This structure forms a timed tree where children are attached at offsets relative to their parent, representing the precise timing of operations.

### Why it exists

The scheduled IR is the canonical real-time experiment representation with concrete timing and offsets, suitable for code generation and diagnostics.

### Where it lives

- Rust crate: `laboneq-ir`
- Source files:
  - [`src/rust/laboneq-ir/src/node.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs)
  - [`src/rust/laboneq-ir/src/ir.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs)
  - [`src/rust/laboneq-ir/src/experiment.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/experiment.rs)

### Who consumes it

- The Rust code generator consumes this IR to produce device-specific code.
- The Python `RealtimeCompiler` wraps this IR for further processing.
- Diagnostic tools and pulse sheet viewers consume this IR.

### Invariants and semantics

- All durations and offsets are in `TinySamples` units (integer sample counts).
- The tree is strictly timed; children offsets are non-negative.
- Section timing modes are absent because timing is resolved.
- Oscillator and precompensation setup nodes are injected at the root.
- Loop preambles and iterations are explicitly represented.

---

## 5. Schedule map and recipe

### What exists

The schedule map and recipe represent the compilation output that encodes the timing, parameter usage, and device configuration for execution.

- The schedule map encodes timing grids, loop chunking, and parameter usage.
- The recipe contains device-specific initialization, waveform maps, command tables, integration weights, and phase increment maps.

These data structures are used to orchestrate execution and replacement flows at runtime.

### Why it exists

They provide a compact, structured representation of the compiled experiment, enabling efficient runtime control, waveform replacement, and feedback.

### Where it lives

- Python package:
  - `src/python/laboneq/data/recipe.py`
  - `src/python/laboneq/data/scheduled_experiment.py`
- Rust crates:
  - `laboneq-scheduler` (for scheduling and recipe generation)
  - `laboneq-codegenerator` (for code generation artifacts)

### Who consumes it

- The controller consumes the recipe and schedule map to execute experiments.
- The code generator produces these artifacts from the scheduled IR.
- The runtime uses these for waveform upload, trigger configuration, and result mapping.

### Invariants and semantics

- The schedule map respects timing grids and chunking constraints.
- Recipes are device-class specific and validated by device implementations.
- Waveform and command-table indices are stable and consistent.

---

## 6. Code generation artifacts

### What exists

Code generation artifacts are the device-specific outputs produced by the code generator from the scheduled IR. For QCCS/SeqC devices, these include:

- SeqC source code and ELF binaries
- Sampled waveform pools and wave indices
- Command tables and pulse-to-waveform maps
- Phase-increment command-table maps
- Integration kernels, weights, and times
- Signal delays and result handle maps
- Feedback register configurations and measurement allocations

### Why it exists

These artifacts are the final compiled outputs that are uploaded to hardware devices and executed in real time.

### Where it lives

- Python package: `src/python/laboneq/compiler/seqc/code_generator.py`
- Rust crate: `laboneq-codegenerator`
- Python Rust extension: `src/python/laboneq/_rust/codegenerator/`

### Who consumes it

- The controller uploads these artifacts to devices.
- The runtime uses them for waveform replacement and feedback.
- Diagnostic tools may inspect these artifacts.

### Invariants and semantics

- Sampled waveforms for SHFQA devices are scaled to be strictly below magnitude 1.0.
- Waveform naming conventions differ for single-channel and IQ signals.
- Artifacts are validated for consistency and compatibility with device capabilities.

---

## 7. Near-time execution IR

### What exists

The near-time execution IR is a Python-side representation of the near-time control flow, separate from the real-time scheduled IR. It consists of statements such as:

- `Sequence` (nested blocks)
- `Nop` (no operation)
- `ExecSet` (node writes)
- `ExecNeartimeCall` (near-time callbacks)
- `SetSoftwareParam` and `SetSoftwareParamLinear` (parameter sweeps)
- `ForLoop` (near-time loops)
- `ExecRT` (real-time boundary marker)

This IR is built by the `ExecutionFactoryFromExperiment` from the Python DSL.

### Why it exists

It enables near-time control flow interpretation, parameter sweeping, and callback invocation in the controller runtime, distinct from the real-time hardware execution.

### Where it lives

- Python package: `src/python/laboneq/executor/`
- Key files:
  - [`executor.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/executor/executor.py)
  - [`execution_from_experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/executor/execution_from_experiment.py)

### Who consumes it

- The `NearTimeRunner` executes this IR asynchronously in the controller.
- Callbacks and parameter updates are handled here.
- The runtime uses this IR to orchestrate real-time experiment runs.

### Invariants and semantics

- `ExecRT` marks the boundary between near-time and real-time execution.
- Near-time loops correspond to Python `Sweep` constructs.
- The IR supports nested sequences and callbacks.

---

## 8. Runtime `RecipeData` and `ScheduledExperiment`

### What exists

`ScheduledExperiment` is the controller-facing compilation product bundling:

- Setup fingerprint
- `Recipe` (device and backend-specific execution plan)
- Backend artifacts (waveforms, command tables, etc.)
- Optional scheduled-event dumps (pulse sheet)
- Near-time execution statement tree
- Real-time loop properties (`RtLoopProperties`)
- Result shape metadata (`ResultShapeInfo`)

`RecipeData` is a runtime IR derived from `ScheduledExperiment` by `recipe_processor.pre_process_compiled()`. It includes:

- `RtExecutionInfo` for real-time execution
- Device-specific recipe data (`DeviceRecipeData`) for HDAWG, UHFQA, SHFQA, SHFSG, SHFPPC
- `AttributeValueTracker` for oscillator frequencies and device attributes
- Execution and result-transfer wait time estimates
- Waveform and command-table preparation helpers

### Why it exists

These runtime IRs bridge the compiled experiment and the concrete LabOne API calls to devices, enabling runtime validation, preparation, and execution orchestration.

### Where it lives

- Python package:
  - `src/python/laboneq/data/scheduled_experiment.py`
  - `src/python/laboneq/controller/recipe_processor.py`
  - `src/python/laboneq/controller/controller.py`

### Who consumes it

- The `Controller` owns and executes `ScheduledExperiment` and `RecipeData`.
- Device classes consume `DeviceRecipeData` for hardware-specific execution.
- The runtime uses these IRs for waveform upload, trigger setup, and result collection.

### Invariants and semantics

- Setup fingerprint ensures experiment and hardware setup consistency.
- Recipe data is validated by device implementations.
- Attribute trackers maintain consistent oscillator and device state.
- Execution timing estimates support asynchronous coordination.

---

## Summary diagram: IR flow and transformations

```mermaid
graph TD
  A[Python DSL Experiment Tree] -->|Payload builder| B[ExperimentInfo]
  B -->|Compatibility bridge| C[Rust DSL Operation Tree]
  C -->|Scheduling passes| D[Scheduled IR Node Tree (IrNode)]
  D -->|Code generation| E[Code Generation Artifacts]
  D -->|Recipe generation| F[Schedule Map & Recipe]
  A -->|Execution factory| G[Near-time Execution IR]
  F -->|Preprocessing| H[Runtime RecipeData]
  H -->|Controller| I[ScheduledExperiment & Runtime Execution]

  style A fill:#f9f,stroke:#333,stroke-width:2px
  style B fill:#bbf,stroke:#333,stroke-width:2px
  style C fill:#bfb,stroke:#333,stroke-width:2px
  style D fill:#ffb,stroke:#333,stroke-width:2px
  style E fill:#fbb,stroke:#333,stroke-width:2px
  style F fill:#bff,stroke:#333,stroke-width:2px
  style G fill:#fbf,stroke:#333,stroke-width:2px
  style H fill:#aff,stroke:#333,stroke-width:2px
  style I fill:#faa,stroke:#333,stroke-width:2px
```

---

## Detailed descriptions

### Python DSL experiment tree

The Python DSL exposes a rich set of classes and methods to define experiments. The root `Experiment` class contains a tree of `Section` objects, each with a unique identifier (`uid`), name, alignment, triggers, and timing mode. Sections can contain operations such as pulse plays, acquisitions, delays, and control flow constructs.

For example, the `Section` class in [`section.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py) provides methods like:

```python
def play(self, pulse: Pulse, signal: Signal, amplitude: float = 1.0, phase: float = 0.0):
    # Append a PlayPulse operation to this section
    ...
```

Real-time loops are represented by `AcquireLoopRt` with fixed execution type `real-time`. Near-time loops are represented by `Sweep` with parameters and chunking options.

The DSL enforces constraints such as:

- Only one `AcquireLoopRt` per experiment
- Near-time sections cannot be nested inside real-time loops

This tree is mutable during experiment construction but becomes immutable once passed to the payload builder.

### ExperimentInfo payload

The `ExperimentInfoBuilder` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py)) traverses the DSL tree and device setup to produce `ExperimentInfo`. This payload contains:

- `SignalInfo` entries mapping logical signals to physical channels
- Calibration data merged from baseline and experiment-specific calibrations
- Parameter information for sweeps and driven parameters
- Chunking information for near-time loops
- Acquire-loop metadata including acquisition type and repetition mode

The builder validates signal conflicts and parameter consistency. It converts calibration fields such as oscillators, LO frequencies, voltage offsets, mixer calibration, precompensation, amplifier pump data, automute, and output routing into compiler input structures.

The resulting `ExperimentInfo` is serialized into Cap'n Proto format for consumption by the Rust compiler.

### Rust DSL operation tree

The Rust DSL tree is defined by the `Operation` enum variants in [`variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs). Each node in the tree is an `ExperimentNode` with a kind and children.

Operations include:

| Operation Kind       | Description                                      |
|----------------------|-------------------------------------------------|
| `Root`               | Root node of the experiment                      |
| `Section`            | Section with alignment, triggers, timing mode   |
| `PrngSetup`          | Pseudo-random number generator setup            |
| `PrngLoop`           | PRNG-based loop                                  |
| `Sweep`              | Near-time parameter sweep                        |
| `AveragingLoop`      | Real-time averaging loop                         |
| `RealTimeBoundary`   | Marks boundary between near-time and real-time  |
| `Match`              | Real-time conditional match                      |
| `Case`               | Case branch in match                             |
| `Reserve`            | Reserved signal constraints                      |
| `PlayPulse`          | Play pulse on signal                             |
| `Acquire`            | Acquire operation                                |
| `Delay`              | Delay operation                                 |
| `ResetOscillatorPhase` | Reset oscillator phase                         |
| `NearTimeCallback`   | Near-time callback placeholder                   |
| `SetNode`            | Set node operation                               |

This IR is strongly typed and normalized, facilitating compiler passes such as validation, rewriting, and scheduling.

### Scheduled IR node tree (`IrNode`)

The scheduled IR is the final real-time timed tree representation. The `IrNode` struct ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs)) contains:

- `kind: IrKind` — the operation kind (enum)
- `length: TinySamples` — duration in sample units
- `children: Vec<NodeChild>` — children with relative offsets

The `IrKind` enum ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs)) includes:

| IrKind Variant           | Description                                   |
|-------------------------|-----------------------------------------------|
| `Root`                  | Root node                                     |
| `Section`               | Section with triggers and PRNG setup          |
| `Loop`                  | Loop with iterations and kind                  |
| `LoopIteration`         | Loop iteration body                            |
| `LoopIterationPreamble` | Loop preamble with oscillator sweep setup     |
| `PlayPulse`             | Play pulse operation                           |
| `Acquire`               | Acquire operation                              |
| `Delay`                 | Delay operation                               |
| `ClearPrecompensation`  | Zero-duration node clearing precompensation   |
| `Match`                 | Real-time match node                           |
| `Case`                  | Case branch node                              |
| `SetOscillatorFrequency`| Set oscillator frequency                       |
| `ChangeOscillatorPhase` | Change oscillator phase                        |
| `PpcStep`               | Pump-probe-cancellation step                   |

The tree is strictly timed with children offsets relative to parents, enabling precise scheduling.

### Schedule map and recipe

The schedule map encodes timing grids, chunking, and parameter usage, while the recipe contains device-specific execution plans.

The Python `ScheduledExperiment` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py)) bundles:

- Setup fingerprint
- `Recipe` object with device configurations
- Backend artifacts (waveforms, command tables)
- Near-time execution tree
- Real-time loop properties
- Result shape metadata

The recipe and schedule map are generated by the Rust scheduler and code generator, then wrapped by Python for runtime use.

### Code generation artifacts

The code generator produces device-specific artifacts such as:

- SeqC source and ELF binaries
- Sampled waveform pools with compression
- Command tables for pulse sequencing
- Integration kernels and weights
- Phase increment maps for feedback
- Signal delay configurations
- Result handle mappings

These are produced by the Rust code generator and exposed via the Python Rust extension (`laboneq._rust.codegenerator`).

### Near-time execution IR

The near-time execution IR is a Python statement tree representing near-time control flow, parameter sweeps, and callbacks.

Key statement types include:

| Statement Type       | Description                          |
|----------------------|------------------------------------|
| `Sequence`           | Nested block of statements          |
| `Nop`                | No operation                       |
| `ExecSet`            | Node write operation                |
| `ExecNeartimeCall`   | Near-time callback invocation       |
| `SetSoftwareParam`   | Set software parameter              |
| `SetSoftwareParamLinear` | Set parameter with linear sweep |
| `ForLoop`            | Near-time loop                     |
| `ExecRT`             | Real-time boundary marker           |

This IR is interpreted by the `NearTimeRunner` in the controller runtime.

### Runtime `RecipeData` and `ScheduledExperiment`

`RecipeData` is derived from `ScheduledExperiment` by preprocessing in the controller. It contains:

- Real-time execution info
- Device-specific recipe data for each instrument type
- Attribute value trackers for oscillators and device state
- Execution timing estimates
- Helpers for waveform and command-table preparation

The `Controller` uses these IRs to orchestrate experiment execution, waveform upload, trigger configuration, and result collection.

---

## Summary table of IRs

| IR / Data Structure       | Layer / Language | Purpose                                    | Location (Key Files)                                                                                   | Consumers                          | Key Invariants / Notes                                  |
|--------------------------|------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------|--------------------------------------------------------|
| Python DSL experiment tree | Python           | User-facing experiment description          | `src/python/laboneq/dsl/experiment/experiment.py`, `section.py`                                       | Payload builder, user code        | Single `AcquireLoopRt`, no near-time in real-time loops |
| `ExperimentInfo`          | Python           | Normalized payload bridging to Rust         | `src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py` | Rust compiler bridge              | Unique signal mapping, merged calibration              |
| Rust DSL operation tree   | Rust             | Normalized experiment tree pre-scheduling   | `src/rust/laboneq-dsl/src/operation/variants.rs`                                                      | Scheduler, lowering passes        | Typed operations, real-time compatibility               |
| Scheduled IR node tree    | Rust             | Timed real-time IR after scheduling          | `src/rust/laboneq-ir/src/node.rs`, `ir.rs`, `experiment.rs`                                           | Code generator, diagnostics       | Timed tree with `TinySamples` offsets                   |
| Schedule map & recipe     | Python / Rust    | Execution plan and timing metadata           | `src/python/laboneq/data/recipe.py`, `scheduled_experiment.py`                                        | Controller, runtime               | Timing grids, chunking, device-specific                 |
| Code generation artifacts | Rust / Python    | Device-specific compiled code and waveforms | `src/python/laboneq/compiler/seqc/code_generator.py`, Rust code generator crates                      | Controller, runtime               | Waveform scaling, naming conventions                    |
| Near-time execution IR    | Python           | Near-time control flow and callbacks         | `src/python/laboneq/executor/executor.py`, `execution_from_experiment.py`                             | NearTimeRunner, controller        | `ExecRT` marks real-time boundary                       |
| Runtime `RecipeData`      | Python           | Runtime execution plan and device configs    | `src/python/laboneq/controller/recipe_processor.py`, `data/scheduled_experiment.py`                   | Controller, device classes        | Setup fingerprint, attribute tracking                  |

---

## References used on this page

1. LabOne Q repository, Python DSL Experiment: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py  
2. LabOne Q repository, Python DSL Section: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py  
3. LabOne Q repository, ExperimentInfoBuilder: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py  
4. LabOne Q repository, Rust DSL operation variants: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs  
5. LabOne Q repository, Rust IR node model: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs  
6. LabOne Q repository, Rust IR definitions: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs  
7. LabOne Q repository, ScheduledExperiment model: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py  
8. LabOne Q repository, Code generator wrapper: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/seqc/code_generator.py  
9. LabOne Q repository, NearTimeRunner: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/near_time_runner.py  
10. LabOne Q repository, Recipe processor: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/recipe_processor.py  
11. LabOne Q repository, Executor IR: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/executor/executor.py  
12. LabOne Q repository, Execution factory: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/executor/execution_from_experiment.py  
13. LabOne Q repository, Controller: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py  
14. LabOne Q repository, Compiler workflow: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py  
15. LabOne Q repository, Compiler compatibility bridge: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py  
16. LabOne Q repository, Rust compiler Python bridge: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs  

---

This concludes the detailed overview of intermediate representations in LabOne Q. Understanding these IRs and their transformations is essential for maintaining and extending the compiler and runtime system.
