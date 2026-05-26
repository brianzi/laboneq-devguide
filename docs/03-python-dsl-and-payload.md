# Python DSL and Compiler Payload

The Python DSL is a construction API for an experiment object graph. It is intentionally more permissive and ergonomic than the compiler's later representations. Users build experiments with context managers for sections, sweeps, acquire loops, matches, and callbacks; operations such as play, delay, acquire, reserve, and call are attached to sections. Logical signals connect those operations to roles in a device setup.

This chapter starts after the frontend mechanics described in [User-facing interfaces and frontend internals](02a-user-facing-interfaces.md). It assumes that Python context managers have already produced an `Experiment`, that setup and calibration information are available, and that the task is now to form compiler input. It stops at the compiler input boundary and does not describe scheduling, AWG grouping, or waveform merging. Those semantics are introduced later.

## Experiment object graph

The core frontend sources are `src/python/laboneq/dsl/experiment/experiment.py` and `src/python/laboneq/dsl/experiment/section.py`. The `Experiment` class owns top-level metadata, signal declarations, calibration, and methods that create nested sections. Section subclasses distinguish ordinary sections, acquire loops, sweeps, PRNG loops, match/case constructs, and related real-time control constructs.

| DSL concept | Compiler-relevant meaning |
| --- | --- |
| Experiment | Root container for signals, calibration, sections, and operations. |
| Section | Nested region with alignment, length, repetition, execution type, and child operations or child sections. |
| Sweep/acquire loop | Loop metadata and parameter bindings that later become timing and near-time/real-time execution structure. |
| Play/acquire/delay/reserve operations | Leaf operations associated with a logical signal and optional pulse/acquisition metadata. |
| Calibration | Logical-signal-level configuration such as oscillator, mixer, amplitude, range, port delay, and device-related constraints. |
| Logical signal | Symbolic lane used by the DSL and setup; it is not yet an AWG-local waveform channel. |

The DSL records **experimental intent**. It can say that two logical readout signals receive overlapping pulses, but it does not yet decide whether those pulses are separate physical waveforms, subchannels of one waveform, or invalid because of oscillator constraints. That decision requires setup, backend, and device-trait information.

## Payload construction

`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py` converts the Python object graph into an `ExperimentInfo`-like compiler input. This conversion is a semantic narrowing step. The builder traverses sections, records operations in compiler-oriented structures, extracts parameters, validates real-time boundaries, converts calibration data, and attaches signal/setup information.

```mermaid
graph TD
    E[Experiment object graph] --> T[Traversal of sections and operations]
    T --> P[Parameter extraction]
    T --> S[Signal and calibration conversion]
    T --> R[Real-time structure validation]
    P --> EI[ExperimentInfo compiler input]
    S --> EI
    R --> EI
```

The important property of this payload is that it is still **not a schedule**. It preserves the experiment's nested structure and enough setup information for compilation, but offsets, resolved section lengths, grid adjustments, AWG-local partitions, and merged waveform intervals are still absent.

## Python-to-Rust compatibility bridge

The workflow files in `src/python/laboneq/compiler/workflow` adapt Python-side data into the forms consumed by the Rust compiler components. `compiler.py` and `realtime_compiler.py` orchestrate compilation from Python, while `compat.py` contains conversion logic for experiment information, setup data, calibration values, precompensation, and parameter stores.

| Boundary | Source files | Information crossing the boundary |
| --- | --- | --- |
| DSL to payload | `experiment_info_builder.py` | Sections, operations, signals, parameters, calibration. |
| Payload to Rust compiler | `compiler/workflow/compat.py`, `realtime_compiler.py` | Normalized experiment/setup/parameter data suitable for Rust preprocessing and scheduling. |
| Rust result to Python | `compiler/seqc/code_generator.py`, `data/scheduled_experiment.py` | Recipe, generated code, waveforms, command tables, result metadata, execution metadata. |

## Compiler settings at the payload boundary

`Session.compile(..., compiler_settings=...)` and `Session.run(..., compiler_settings=...)` pass a user-supplied dictionary into the compilation workflow. This dictionary is intentionally not interpreted by the DSL layer. Its first significant normalization happens near the Python-to-Rust compatibility bridge, where `compiler/workflow/compat.py` sanitizes selected values before passing the remaining dictionary to the Rust compiler representation builder.

```mermaid
graph TD
    A[Session.compile or compile_experiment] --> B[Compiler workflow]
    B --> C[Python CompilerSettings subset]
    B --> D[Payload and setup compatibility conversion]
    D --> E[Sanitize compiler_settings dictionary]
    E --> F[Rust experiment builder]
    F --> G[Realtime compiler, scheduler, linker, codegen]
    G --> H[Reporter and ScheduledExperiment output]
```

The Python dataclass currently models only the settings consumed directly by Python-side code: `LOG_REPORT` and `IGNORE_RESOURCE_LIMITATION_ERRORS`. The compiler settings file explicitly notes that the remaining settings are processed in the Rust compiler. This split is important when reading the pipeline: some settings affect Python reporting or post-check behaviour, while others are forwarded after light validation and change lower-level compilation behaviour.

| Setting or setting family | First visible handling point | Effect in the pipeline |
| --- | --- | --- |
| `LOG_REPORT` | Python `CompilerSettings` and compilation reporter. | Controls whether the post-compilation report is logged. |
| `IGNORE_RESOURCE_LIMITATION_ERRORS` | Python resource-usage handling. | Allows compilation to continue past resource-limit reports that would otherwise be treated as errors. |
| `MAX_EVENTS_TO_PUBLISH` | `compat._sanitize_compiler_settings`. | Converted from float to integer if needed; warned as ineffective unless `OUTPUT_EXTRAS=True` is also set. |
| `OUTPUT_EXTRAS` | Forwarded to lower compiler stages. | Enables extra schedule metadata used by diagnostics such as the pulse sheet viewer. |
| `AMPLITUDE_RESOLUTION_BITS` and `PHASE_RESOLUTION_BITS` | `compat._sanitize_compiler_settings`. | Clamped to non-negative integers before being forwarded. |
| Deprecated or inert command-table hints | `compat._sanitize_compiler_settings`. | Warnings are emitted and no-effect settings may be removed before forwarding. |

The pulse-sheet viewer is a concrete example of these settings in practice. If a compiled experiment lacks the schedule extras needed for rendering, the viewer recompiles with `OUTPUT_EXTRAS=True`, disables normal report logging, and raises `MAX_EVENTS_TO_PUBLISH` so that event-list metadata is available for the HTML view. This does not mean the DSL changed; it means the compiler was asked to publish additional diagnostic output.

Later compilation chapters should treat individual settings at the stage where they first alter behaviour. Settings that affect schedule metadata belong near schedule and reporting. Settings that affect waveform or command-table generation belong near code generation. Settings that affect resource-limit handling belong near backend resource mapping and resource usage. Keeping this discussion distributed avoids implying that `compiler_settings` is a single monolithic preprocessor switch.

## Invariants handed to scheduling

By the time the scheduler receives the experiment, the compiler should have a normalized description of the experiment hierarchy, signal names, operation metadata, and relevant setup/calibration information. The scheduler can then reason about timing constraints without needing the original Python context-manager machinery.

The payload layer should not be read as a hidden hardware compiler. It does not perform the interval compaction that merges logical signals onto physical waveforms. Its purpose is to make the Python experiment explicit enough that later stages can reason about it deterministically.
