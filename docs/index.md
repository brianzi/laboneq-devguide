# LabOne Q Developer Guide

This guide is an **unofficial maintainer-oriented map** of the `zhinst/laboneq` code base. Its main purpose is to explain how a Python experiment description becomes scheduled, device-specific control artifacts for Zurich Instruments quantum-control hardware. The guide is organized as a lowering story rather than as a package-by-package tour: each chapter describes one semantic domain, the information added in that domain, and the invariants handed to the next domain.

The most important distinction is that LabOne Q does not compile a DSL program directly into a waveform file. It progressively changes representation. A logical pulse first belongs to a Python experiment tree. Later it belongs to a compiler-input description with setup and calibration metadata. Later still it becomes part of a globally timed schedule. Only after scheduling does the code generator partition the experiment by AWG core, group logical signals that share a physical resource, detect and compact overlapping pulse intervals, and replace logical pulse operations with physical `PlayWave` events.

```mermaid
graph TD
    A[Python experiment DSL\nlogical sections, signals, pulses] --> B[ExperimentInfo / compiler input\nsetup, calibration, parameters]
    B --> C[Rust DSL operation tree\nnormalized experiment operations]
    C --> D[ScheduledNode tree\nglobal timing solution]
    D --> E[shared IrNode tree\nmutable codegen carrier]
    E --> F[backend resource model\nphysical channels, AWG cores, device traits]
    F --> G[AWG-local IrNode trees\nfanout by sequencer core]
    G --> H[virtual signals\nlogical signals sharing one waveform resource]
    H --> I[PlayWave lowering\ninterval compaction and waveform signatures]
    I --> J[SeqC, waves, command tables, recipe]
    J --> K[controller runtime\nupload, execute, acquire, return results]
```

## Reading order

The guide is designed to be read linearly through the compiler chapters. The early chapters give enough repository and hardware context to make source browsing practical; the middle chapters explain the IR transitions; the later chapters explain generated artifacts and runtime execution.

| Chapter | Focus | Main boundary clarified |
| --- | --- | --- |
| [Mental model](01-mental-model.md) | Vocabulary and semantic domains | Logical experiment language versus physical resources. |
| [Repository and build map](02-repository-and-build-map.md) | Source-tree orientation | Python packages, Rust crates, generated extension modules, and dependency roles. |
| [Python DSL and compiler payload](03-python-dsl-and-payload.md) | DSL object graph and `ExperimentInfo` | User-facing construction versus compiler input serialization. |
| [Global scheduling](04-global-scheduling.md) | Section timing, grids, offsets, repetition | Timing solution versus waveform/resource lowering. |
| [IR semantics](05-ir-semantics.md) | Rust operation tree, `ScheduledNode`, `IrNode` | Conceptual differences between similarly named node structures. |
| [Backend resource mapping](06-backend-resource-mapping.md) | Setup-to-device mapping | Logical signals versus physical channels and AWG cores. |
| [AWG-local lowering](07-awg-local-lowering.md) | Multiplexing and interval compaction | Logical `PlayPulse` events versus physical `PlayWave` events. |
| [Code generation artifacts](08-code-generation-artifacts.md) | SeqC, waveforms, command tables, recipe | Intermediate events versus uploadable runtime artifacts. |
| [Runtime controller](09-runtime-controller.md) | Execution and result collection | Compiled experiment versus asynchronous device actions. |
| [Ecosystem and hardware context](10-ecosystem-and-hardware.md) | Related repos and device constraints | LabOne Q relative to toolkit, comms, examples, and manuals. |
| [Extension and maintenance guide](11-extension-and-maintenance.md) | Practical change points | Where to change the compiler safely. |

## Key terminology

| Term | Meaning in this guide |
| --- | --- |
| **Logical signal** | A DSL/setup identifier such as a qubit drive or readout line. It describes an experimental role and calibration, not necessarily a unique physical waveform stream. |
| **Physical channel or port** | An instrument resource named by device setup connections and calibration. Several logical signals may share, couple through, or be constrained by one physical resource. |
| **AWG core** | The sequencer plus waveform-player resource that runs one program and controls one or more outputs, depending on device family and grouping mode. |
| **Scheduling** | The global timing pass that resolves section lengths, offsets, grids, alignment, loops, and repetition semantics. Scheduling is not the same as multiplexing logical signals onto physical waveform streams. |
| **Virtual signal** | A code-generator grouping of logical signals that must be handled together for one AWG-local waveform/event stream. This is the bridge between logical pulses and physical `playWave` operations. |
| **Playwave lowering** | The code-generation stage that collects logical pulse slots for a virtual signal, computes interval cut points, compacts intervals, creates waveform signatures, emits `PlayWave` nodes, and replaces source `PlayPulse` nodes with `Nop`. |

## Maintainer orientation

The guide intentionally separates **semantic representation** from **implementation container**. The word “node” appears in multiple source modules, but the nodes do not all mean the same thing. A scheduler node exists to solve global timing constraints. A shared IR node exists to carry a timed tree through code-generator mutations. An AWG-local IR node exists after fanout to one sequencer core. Treating all of these as “the IR” is precisely what makes the code base hard to read.

The compiler chapters therefore use a stricter vocabulary: a representation is named by the stage that owns its invariants. When a representation crosses from one stage to another, the chapter identifies the conversion source file and the semantic information that is preserved, transformed, or newly introduced.
