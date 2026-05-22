# Glossary

This glossary provides a comprehensive reference for terminology and key concepts encountered in the LabOne Q (`zhinst/laboneq`) codebase and its ecosystem. It covers terms related to the Quantum Control and Computing Stack (QCCS), LabOne Q's Python DSL frontend, compiler and runtime internals, hardware abstractions, and related Zurich Instruments software components. The goal is to orient developers to the terminology, implementation layers, integration boundaries, and invariants used throughout the codebase.

---

## Maintainer orientation

This glossary is organized into thematic sections reflecting the layered architecture of LabOne Q and its ecosystem. Each entry defines a term or concept, explains its role and rationale, and references relevant source files or modules for deeper inspection. When maintaining or extending the codebase, this page helps clarify the meaning and boundaries of key abstractions, avoiding confusion between user-facing DSL constructs, internal intermediate representations (IRs), compiler passes, runtime models, and hardware-layer interfaces.

Developers should use this glossary alongside the repository map (`02-repository-map.md`) and compiler overview (`06-compiler-overview.md`) to understand how terms relate to code locations and workflows. Source links point to stable GitHub paths for direct code reference.

---

## Table of Contents

- [QCCS and Hardware Terms](#qccs-and-hardware-terms)
- [LabOne Q Python DSL Terms](#labone-q-python-dsl-terms)
- [Compiler and Intermediate Representations (IR)](#compiler-and-intermediate-representations-ir)
- [Runtime and Controller Terms](#runtime-and-controller-terms)
- [Zurich Instruments Software Ecosystem](#zurich-instruments-software-ecosystem)

---

## QCCS and Hardware Terms

| Term | Description | Location / Source | Consumers | Invariants / Notes |
|-------|-------------|-------------------|-----------|--------------------|
| **QCCS (Quantum Control and Computing Stack)** | The hardware and software stack provided by Zurich Instruments for quantum experiments. It includes synchronization instruments (PQSC), arbitrary waveform generators (HDAWG, SHFSG), quantum analyzers (SHFQA, UHFQA), and control software. | See [LabOne Q README](https://github.com/zhinst/laboneq/blob/main/README.md), PQSC manual [PQSC manual](https://docs.zhinst.com/pqsc_user_manual/functional_overview.html) | LabOne Q compiler and runtime, hardware devices | QCCS hardware requires globally synchronized programs with sub-nanosecond precision timing; LabOne Q compiler produces sample-precise code for QCCS devices. |
| **PQSC (Programmable Quantum System Controller)** | The central synchronization and control instrument in QCCS. Provides clock distribution, trigger routing, and low-latency data interfaces via ZSync ports. | PQSC manual [PQSC manual](https://docs.zhinst.com/pqsc_user_manual/functional_overview.html) | LabOne Q runtime, device layer | Ensures sub-nanosecond synchronization; LabOne Q runtime uses PQSC for trigger coordination. |
| **ZSync** | Zurich Instruments synchronization protocol and hardware interface for clock and trigger distribution among QCCS devices. | PQSC manual, LabOne Q Core manual | Runtime device communication, synchronization | ZSync latency < 100 ns; critical for timing guarantees in compiled experiments. |
| **HDAWG (High-Density Arbitrary Waveform Generator)** | A Zurich Instruments AWG device used for generating control pulses. | Device-specific code in `src/python/laboneq/controller/devices/device_hdawg.py` | Compiler code generation, runtime execution | Supports waveform upload, trigger control, and advanced sequencer features. |
| **SHFSG (Superconducting High-Frequency Signal Generator)** | Microwave signal generator with multiple output channels and advanced sequencer capabilities. | Device-specific code in `src/python/laboneq/controller/devices/device_shfsg.py` | Compiler code generation, runtime execution | Supports arbitrary waveform generation, branching, command tables, and digital modulation. |
| **SHFQA (Superconducting High-Frequency Quantum Analyzer)** | Combines readout and signal generation channels with integrated measurement units and multistate discrimination. | Device-specific code in `src/python/laboneq/controller/devices/device_shfqa.py` | Compiler code generation, runtime execution | Supports complex sampled pulses, integration kernels, and real-time feedback. |
| **UHFQA (Ultra-High-Frequency Quantum Analyzer)** | Quantum analyzer device similar to SHFQA but operating at ultra-high frequencies. | Device-specific code in `src/python/laboneq/controller/devices/device_uhfqa.py` | Compiler code generation, runtime execution | Supports readout and integration with real-time feedback. |
| **SHFQC (Superconducting High-Frequency Quantum Controller)** | Device combining quantum analyzer and signal generator channels with advanced sequencing and measurement capabilities. | Device-specific code in `src/python/laboneq/controller/devices/device_shfqc.py` | Compiler code generation, runtime execution | Supports multichannel readout and control with integrated sequencing. |
| **ZQCS (Zurich Quantum Control System)** | A legacy or alternative control system not supported by the QCCS backend in LabOne Q. | Backend preprocessing code in `src/rust/laboneq-qccs-backend/src/preprocessor.rs` | Compiler backend | QCCS backend rejects ZQCS setups; LabOne Q targets QCCS hardware. |
| **AWG Device / AWG Key** | Logical abstraction of an AWG device in the compiler backend, identified by stable keys grouping channels and device UIDs. | `src/rust/laboneq-qccs-backend/src/preprocessor.rs` | Compiler backend, scheduler, code generator | Stable keys ensure consistent mapping of signals to hardware AWGs. |
| **Lead Delay** | Per-device or per-signal delay compensation to align timing across hardware channels. | Backend preprocessing, device setup | Compiler backend, runtime device layer | Lead delays are critical for timing alignment and are validated during preprocessing. |
| **Trigger Chain** | The sequence of hardware triggers and synchronization signals used to coordinate device execution. | Runtime device layer, e.g. `src/python/laboneq/controller/devices/device_collection.py` | Runtime controller, devices | Trigger chains are configured to ensure correct start/stop and synchronization of experiments. |

---

## LabOne Q Python DSL Terms

| Term | Description | Location / Source | Consumers | Invariants / Notes |
|-------|-------------|-------------------|-----------|--------------------|
| **Experiment (DSL)** | The main user-facing container representing a quantum experiment. It is setup-independent and defines the pulse sequence and dynamic process. | `src/python/laboneq/dsl/experiment/experiment.py` | Users, payload builder, compiler frontend | Contains sections, sweeps, calibration, and operations; immutable after build phase. |
| **Section** | A structural DSL node grouping operations with timing alignment, triggers, and optional execution types. | `src/python/laboneq/dsl/experiment/section.py` | Experiment, compiler frontend | Sections have unique IDs (`uid`), alignment constraints, and timing modes; nested sections are allowed with constraints. |
| **AcquireLoopRt** | A real-time averaging/acquisition loop boundary in the DSL. | `src/python/laboneq/dsl/experiment/section.py` | Experiment, compiler frontend | Defines acquisition type, averaging mode, repetition mode, and oscillator phase reset; only one per experiment allowed. |
| **Sweep** | A near-time software parameter loop over one or more parameters, supporting chunking and automatic chunk count. | `src/python/laboneq/dsl/experiment/experiment.py` | Experiment, near-time execution | Sweeps drive parameter variation and chunking for experiment repetition. |
| **Match / Case** | Real-time conditional control constructs for branching based on feedback or measurement results. | `src/python/laboneq/dsl/experiment/experiment.py` | Experiment, compiler frontend, scheduler | Used for feedback and conditional execution; resolved during scheduling. |
| **PRNGSetup / PRNGLoop** | Pseudo-random number generator setup and loops for randomized experiment control. | `src/python/laboneq/dsl/experiment/experiment.py` | Experiment, compiler frontend | Provide deterministic randomization for experiments; timing and seed management are critical. |
| **Operation** | Base class for DSL operations such as `PlayPulse`, `Acquire`, `Delay`, `ResetOscillatorPhase`, etc. | `src/python/laboneq/dsl/experiment/operation.py` | Experiment, compiler frontend | Operations carry semantic payloads and are leaf nodes in the DSL tree. |
| **Pulse** | A reusable waveform or pulse definition used in `PlayPulse` operations. | `src/python/laboneq/dsl/experiment/pulse.py` | Experiment, compiler frontend, code generator | Pulses define waveform shapes, parameters, and markers; stored in pulse libraries. |
| **Calibration** | Metadata and parameters describing device calibration such as mixer calibration, amplifier pump, precompensation, and oscillator frequencies. | `src/python/laboneq/dsl/calibration/` | Payload builder, compiler backend | Calibration data is merged with experiment signals to produce device setup. |
| **DeviceSetup** | DSL representation of instruments, wiring, and logical signal lines. | `src/python/laboneq/dsl/device/device_setup.py` | Payload builder, compiler backend, runtime | DeviceSetup is the authoritative description of hardware configuration for compilation and execution. |

---

## Compiler and Intermediate Representations (IR)

| Term | Description | Location / Source | Consumers | Invariants / Notes |
|-------|-------------|-------------------|-----------|--------------------|
| **ExperimentInfo** | A Python data structure produced by `ExperimentInfoBuilder` that lowers the DSL and device setup into a normalized compilation input. | `src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py` | Compiler frontend, compatibility bridge | Contains resolved signals, calibration, parameters, and chunking info; validated for conflicts and consistency. |
| **Compiler Backend** | The Rust backend implementing device-specific compilation logic, preprocessing, scheduling, and lowering to IR. | `src/rust/laboneq-qccs-backend/` | Python compiler workflow, scheduler | Implements `PreprocessedBackendData` trait; validates device combinations and prepares backend metadata. |
| **Experiment (Rust DSL)** | Normalized tree of `ExperimentNode` objects representing the experiment in Rust DSL form after compatibility bridge. | `src/rust/laboneq-dsl/src/operation/variants.rs` | Scheduler, lowering passes | Contains structural nodes (sections, loops, matches) and pulse-level nodes; enriched with semantic fields. |
| **ScheduledNode** | Scheduler-internal representation of scheduled experiment nodes with timing constraints and unresolved parameters. | `src/rust/laboneq-scheduler/src/scheduler.rs` | Scheduler passes, lowering | Intermediate form before final IR; used for timing calculation and validation. |
| **IrNode / IrKind** | The final timed IR node tree representing the scheduled experiment with concrete offsets and durations. | `src/rust/laboneq-ir/src/node.rs`, `ir.rs` | Code generator, diagnostics | Timed tree with children attached at offsets; leaf nodes represent signal-level operations. |
| **ExperimentIr** | The Rust container bundling the root IR node with acquisition type, parameter store, pulse definitions, and device setup. | `src/rust/laboneq-ir/src/experiment.rs` | Code generator, Python compiler bridge | Uses `Arc` for shared ownership; preserves context for code generation and diagnostics. |
| **PulseSheetSchedule** | Data structure representing scheduled pulse events for visualization and emulation. | `src/rust/laboneq-ir/src/pulse_sheet_schedule.rs` | Pulse sheet viewer, diagnostics | Contains section info and event lists for sample-precise pulse visualization. |
| **ScheduleResult** | The output of the scheduling process including the IR, used parameters, and optional pulse sheet schedule. | `src/rust/laboneq-compiler-py/src/lib.rs` | Python compiler workflow | Returned to Python for code generation and further processing. |
| **RealtimeCompiler** | Python orchestration class wrapping Rust compilation and scheduling calls. | `src/python/laboneq/compiler/workflow/realtime_compiler.py` | Compiler workflow | Selects device-specific hooks, calls scheduler, and code generator; manages chunking. |
| **CodeGenerator** | Python wrapper around Rust code generation producing SeqC source, waveforms, command tables, and artifacts. | `src/python/laboneq/compiler/seqc/code_generator.py` | Compiler workflow, runtime | Converts IR to device-specific code and data structures for runtime upload. |
| **SeqC** | The Zurich Instruments sequencer language used for AWG programming. | Compiler code generator output | Runtime device upload | Generated source code and ELF binaries for AWG devices. |

---

## Runtime and Controller Terms

| Term | Description | Location / Source | Consumers | Invariants / Notes |
|-------|-------------|-------------------|-----------|--------------------|
| **ScheduledExperiment** | The controller-facing compilation product bundling setup fingerprint, recipe, artifacts, near-time execution tree, and result metadata. | `src/python/laboneq/data/scheduled_experiment.py` | Controller, runtime | Validated against device setup; immutable during execution. |
| **RecipeData** | Runtime IR repackaging `ScheduledExperiment` for device validation, AWG config extraction, and waveform preparation. | `src/python/laboneq/controller/recipe_processor.py` | Controller, devices | Contains `RtExecutionInfo`, device-specific recipe data, and attribute trackers. |
| **Controller** | The runtime component managing device communication, asynchronous execution, result collection, and experiment orchestration. | `src/python/laboneq/controller/controller.py` | User-facing runtime API | Owns `DeviceCollection`, manages callbacks, and enforces setup consistency. |
| **DeviceCollection** | Abstraction managing multiple connected devices and their topology. | `src/python/laboneq/controller/devices/device_collection.py` | Controller, devices | Coordinates device setup, execution, and result retrieval. |
| **DeviceZI / DeviceBase** | Abstract base classes for hardware devices implementing setup validation, recipe validation, artifact preparation, and execution hooks. | `src/python/laboneq/controller/devices/device_*.py` | Controller, runtime | Concrete subclasses implement device-specific logic using `zhinst.core` and `zhinst.comms`. |
| **NearTimeRunner** | Executor subclass managing near-time loop execution, parameter sweeping, and callback invocation. | `src/python/laboneq/controller/near_time_runner.py` | Controller runtime | Converts near-time IR into step execution; tracks nested loop indices. |
| **Execution IR (Near-Time)** | Python-side IR for near-time control including sequences, loops, parameter sets, and real-time boundaries. | `src/python/laboneq/executor/executor.py` | NearTimeRunner, Controller | Separate from real-time IR; used for software-driven experiment control. |
| **SweepParamsTracker** | Utility tracking parameter values and updates during near-time execution. | `src/python/laboneq/controller/utilities/sweep_params_tracker.py` | NearTimeRunner, Controller | Ensures consistent parameter propagation to compiled recipe. |
| **ResultsBuilder** | Data structure for collecting and organizing experiment results during runtime. | `src/python/laboneq/controller/results.py` | Controller, user | Preallocates buffers and maps hardware result sources to handles. |
| **Callbacks** | User-registered functions invoked during near-time execution for dynamic control or data processing. | `src/python/laboneq/controller/controller.py` | User, runtime | Callback results are stored in experiment results; support asynchronous execution. |

---

## Zurich Instruments Software Ecosystem

| Term | Description | Location / Source | Consumers | Invariants / Notes |
|-------|-------------|-------------------|-----------|--------------------|
| **LabOne Q** | The quantum experiment programming framework built on top of QCCS hardware, providing a Python DSL, compiler, and runtime. | `https://github.com/zhinst/laboneq` | Quantum experiment developers | Separates experiment definition, compilation, and execution; supports real-time and near-time control. |
| **zhinst-core** | Native Python API for communicating with Zurich Instruments devices. | PyPI: [zhinst-core](https://pypi.org/project/zhinst-core/) | LabOne Q runtime, toolkit | Binary extension wrapping LabOne API; low-level device communication. |
| **zhinst-comms** | Protocol stack for communicating with LabOne data server; not intended for direct use. | PyPI: [zhinst-comms](https://pypi.org/project/zhinst-comms/) | zhinst-core, LabOne Q runtime | Provides network protocol layers for device communication. |
| **zhinst-toolkit** | High-level Python driver package built on `zhinst.core` providing pythonic device control. | `https://github.com/zhinst/zhinst-toolkit` | LabOne Q runtime, users | Runtime dependency of LabOne Q; abstracts device control and session management. |
| **LabOne API examples** | Repository containing low-level examples for controlling Zurich Instruments devices via LabOne APIs. | `https://github.com/zhinst/labone-api-examples` | Developers learning device APIs | Contrasts with LabOne Q's higher-level DSL approach. |
| **LabOne Q Applications** | Application-layer package built on LabOne Q providing domain-level experiment libraries and calibrations. | `https://github.com/zhinst/laboneq-applications` | LabOne Q users and developers | Consumes LabOne Q DSL and runtime; provides reusable quantum experiment components. |

---

## Selected Mermaid Diagram: LabOne Q Compilation and Runtime Layers

```mermaid
graph TD
  User[User Software]
  DSL[Python DSL (Experiment, Section, Operations)]
  Payload[Payload Builder (ExperimentInfo)]
  Bridge[Python/Rust Compatibility Bridge]
  RustDSL[Rust DSL (ExperimentNode Tree)]
  Scheduler[Scheduler (Scheduling, Validation)]
  IR[Intermediate Representation (IrNode Tree)]
  CodeGen[Code Generator (SeqC, Waveforms)]
  Runtime[Runtime Controller (ScheduledExperiment, RecipeData)]
  Devices[Device Layer (DeviceCollection, DeviceZI)]
  Hardware[QCCS Hardware (PQSC, HDAWG, SHFSG, SHFQA, UHFQA)]

  User --> DSL
  DSL --> Payload
  Payload --> Bridge
  Bridge --> RustDSL
  RustDSL --> Scheduler
  Scheduler --> IR
  IR --> CodeGen
  CodeGen --> Runtime
  Runtime --> Devices
  Devices --> Hardware
```

---

## Summary

This glossary captures the layered abstractions and terminology essential for understanding and maintaining the LabOne Q codebase. It distinguishes user-facing DSL constructs from internal IRs and runtime models, clarifies hardware and backend-specific concepts, and situates LabOne Q within the Zurich Instruments software ecosystem. Developers should refer to the linked source files and documentation for detailed implementation and usage patterns.

---

## References used on this page

1. LabOne Q repository and README: https://github.com/zhinst/laboneq  
2. LabOne Q Core manual: https://docs.zhinst.com/labone_q_user_manual/core/index.html  
3. PQSC manual: https://docs.zhinst.com/pqsc_user_manual/functional_overview.html  
4. SHFQC manual: https://docs.zhinst.com/shfqc_user_manual/functional_overview.html  
5. SHFSG manual: https://docs.zhinst.com/shfsg_user_manual/functional_overview.html  
6. zhinst-comms PyPI: https://pypi.org/project/zhinst-comms/  
7. zhinst-core PyPI: https://pypi.org/project/zhinst-core/  
8. zhinst-toolkit GitHub: https://github.com/zhinst/zhinst-toolkit  
9. Python DSL Experiment: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py  
10. Python DSL Section: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py  
11. ExperimentInfoBuilder: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py  
12. Compiler workflow: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py  
13. Compiler compatibility bridge: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py  
14. Realtime compiler: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/realtime_compiler.py  
15. Rust IR crate: https://github.com/zhinst/laboneq/tree/main/src/rust/laboneq-ir  
16. Rust IR definitions: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs  
17. Rust DSL operation variants: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs  
18. Rust scheduler: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs  
19. Rust lowering pass: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs  
20. QCCS backend preprocessing: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs  
21. Code generation boundary and outputs: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/seqc/code_generator.py  
22. Runtime and controller execution: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py  
23. NearTimeRunner: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/near_time_runner.py  
24. ScheduledExperiment model: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py  
25. LabOne Q user manual: https://docs.zhinst.com/labone_q_user_manual/  
26. LabOne API examples: https://github.com/zhinst/labone-api-examples  
27. LabOne Q Applications: https://github.com/zhinst/laboneq-applications
