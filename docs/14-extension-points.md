# Extension and maintenance guide

This page provides a comprehensive orientation for developers extending and maintaining the LabOne Q (`zhinst/laboneq`) codebase. It explains where and how to add new domain-specific language (DSL) operations, compiler passes, device support, calibration fields, artifact types, tests, and compatibility shims. It also offers a debugging and maintenance checklist to ensure code quality and system invariants. This guide is intended for maintainers and contributors who want to understand the extension points and internal invariants of the LabOne Q compiler and runtime system.

---

## How to read this page as a maintainer

This page is structured to first orient you on the major extension points in the LabOne Q architecture, then provide detailed guidance on each area. It assumes familiarity with the overall LabOne Q architecture as described in earlier guide pages, especially the DSL frontend, compiler workflow, IR models, Rust compiler core, scheduling, code generation, runtime controller, and device layers.

Each section answers the following questions:

- **What exists?** The abstraction or component currently implemented.
- **Rationale**: The design rationale or role in the system.
- **Source Location**: The relevant source directories and files.
- **Who consumes it?** The clients or downstream components.
- **What invariants it carries?** Important correctness or semantic guarantees.

Code and file paths are given as relative to the repository root, with links to the GitHub source for quick reference. Where details are inferred from source inspection rather than explicit documentation, this is noted.

---

## Table of contents

- [DSL operations](#dsl-operations)
- [Compiler passes](#compiler-passes)
- [Device support](#device-support)
- [Calibration fields](#calibration-fields)
- [Artifact types](#artifact-types)
- [Tests and testing](#tests-and-testing)
- [Compatibility shims](#compatibility-shims)
- [Debugging and maintenance checklist](#debugging-and-maintenance-checklist)

---

## DSL operations

### What exists?

The LabOne Q Python DSL frontend provides a rich set of experiment-building operations, structured as Python classes and methods. The main user-facing container is the `Experiment` class (`src/python/laboneq/dsl/experiment/experiment.py`), which holds a tree of `Section` objects (`src/python/laboneq/dsl/experiment/section.py`) and their child operations.

Operations include:

- Pulse plays (`play()`)
- Acquisitions (`acquire()`, `measure()`)
- Delays (`delay()`)
- Resets (`reset_oscillator_phase_()`)
- Control flow constructs (`Sweep`, `Match`, `Case`, `PRNGSetup`, `PRNGLoop`)
- Node writes (`set_node()`)

These operations are implemented as Python classes in the `laboneq.dsl.experiment` package, with concrete subclasses for each operation kind.

Rationale

The DSL abstracts the complex hardware control into a user-friendly, composable Python API. It allows experimenters to declaratively specify pulse sequences, loops, conditional branches, and measurement instructions without dealing with low-level device details.

Source Location

- Python DSL experiment root: `src/python/laboneq/dsl/experiment/experiment.py`
- Section and operation classes: `src/python/laboneq/dsl/experiment/section.py` and sibling files
- DSL calibration: `src/python/laboneq/dsl/calibration/`
- Device definitions: `src/python/laboneq/dsl/device/instruments/`

### Who consumes it?

- The `ExperimentInfoBuilder` (`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py`) lowers the DSL tree into the intermediate `ExperimentInfo` data structure.
- The compiler workflow (`src/python/laboneq/compiler/workflow/compiler.py`) consumes `ExperimentInfo` to produce the compiled experiment.
- The runtime controller consumes the compiled experiment for execution.

### What invariants it carries?

- The DSL tree must be well-formed: e.g., at most one real-time acquisition loop (`AcquireLoopRt`), no nested real-time loops inside near-time loops.
- Timing constraints are enforced later in the compiler but the DSL should not violate structural rules.
- Operations carry unique identifiers (`uid`) for traceability.

### How to add new DSL operations?

1. Define a new Python class in the `laboneq.dsl.experiment` package, subclassing the appropriate base operation class.
2. Add mutating helper methods on `Section` or `Experiment` to create and append the new operation.
3. Extend the `ExperimentInfoBuilder` to recognize and lower the new operation into the payload.
4. Add corresponding Rust DSL operation variants in the `laboneq-dsl` crate (`src/rust/laboneq-dsl/src/operation/variants.rs`).
5. Update the lowering passes to handle the new operation.

---

## Compiler passes

### What exists?

The LabOne Q compiler is a multi-stage pipeline orchestrated primarily in Python but backed by Rust for performance-critical passes.

Key components:

- Python orchestration: `src/python/laboneq/compiler/workflow/compiler.py`
- Compatibility bridge: `src/python/laboneq/compiler/workflow/compat.py`
- Real-time compiler wrapper: `src/python/laboneq/compiler/workflow/realtime_compiler.py`
- Rust compiler bridge: `src/rust/laboneq-compiler-py/src/lib.rs`
- Rust IR crate: `src/rust/laboneq-ir/`
- Rust scheduler: `src/rust/laboneq-scheduler/`
- Lowering passes: `src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`

Rationale

The compiler transforms the high-level DSL experiment into a timed, scheduled intermediate representation (IR) suitable for code generation and device execution. It enforces timing constraints, resolves parameters, and applies hardware-specific preprocessing.

Source Location

- Python compiler workflow and hooks: `src/python/laboneq/compiler/workflow/`
- Rust IR and scheduler crates: `src/rust/laboneq-ir/`, `src/rust/laboneq-scheduler/`
- Backend-specific preprocessing: `src/rust/laboneq-qccs-backend/`

### Who consumes it?

- The code generator consumes the scheduled IR to produce device-specific code.
- The runtime controller consumes the compiled experiment artifacts.
- Developers extending the compiler add passes here.

### What invariants it carries?

- The IR is a timed tree with concrete sample offsets.
- Scheduler passes ensure signal sampling-rate commensurability.
- Loop unrolling and parameter resolution must preserve experiment semantics.
- Validation passes reject unsupported constructs or timing violations.

### How to add new compiler passes?

1. Implement the pass in Rust within the `laboneq-scheduler` crate, following existing pass patterns.
2. Register the pass in the scheduler pipeline (`src/rust/laboneq-scheduler/src/scheduler.rs`).
3. Expose any necessary hooks in the Python compiler workflow.
4. Add tests validating the pass behavior.

---

## Device support

### What exists?

Device support is implemented as a set of Python classes representing instruments and their capabilities, with corresponding Rust backend support.

Key device classes:

- `DeviceZI` base class and subclasses for specific instruments (`device_hdawg.py`, `device_shfqa.py`, `device_shfsg.py`, `device_uhfqa.py`, `device_pqsc.py`, etc.) located in `src/python/laboneq/controller/devices/`
- Device setup and calibration helpers in `src/python/laboneq/dsl/device/`
- Backend preprocessing for QCCS devices in Rust (`src/rust/laboneq-qccs-backend/`)

Rationale

To abstract hardware-specific details such as trigger chains, waveform upload, and device capabilities, enabling the compiler and runtime to target multiple instruments seamlessly.

Source Location

- Python device abstractions: `src/python/laboneq/controller/devices/`
- Device setup DSL: `src/python/laboneq/dsl/device/`
- Rust backend preprocessing: `src/rust/laboneq-qccs-backend/`

### Who consumes it?

- The compiler queries device capabilities and setup during compilation.
- The runtime controller uses device classes to upload artifacts, arm devices, and collect results.
- Developers adding support for new instruments extend these classes.

### What invariants it carries?

- Device classes must validate setup consistency.
- Device-specific recipe data must be consistent with compiled artifacts.
- Emulation flags and runtime checks ensure safe operation.

### How to add new device support?

1. Add a new device class in `src/python/laboneq/controller/devices/`, subclassing `DeviceZI` or the appropriate base.
2. Define device-specific setup, calibration, and recipe validation.
3. Extend the DSL device definitions if needed.
4. Add Rust backend preprocessing if the device requires special compilation handling.
5. Add tests for setup, compilation, and runtime execution.

---

## Calibration fields

### What exists?

Calibration data is represented in the DSL calibration package (`src/python/laboneq/dsl/calibration/`) and includes:

- Oscillator calibration
- Mixer calibration
- Output routing
- Precompensation filters
- Amplifier pump calibration
- Signal calibration

Calibration fields are merged from baseline setup and experiment-level overrides during payload building.

Rationale

Calibration ensures that the physical hardware behaves as expected, compensating for imperfections and enabling precise pulse shaping and measurement.

Source Location

- DSL calibration: `src/python/laboneq/dsl/calibration/`
- Payload builder merges calibration: `src/python/laboneq/implementation/payload_builder/experiment_info_builder/`
- Calibration data structures in Rust are serialized via Cap'n Proto schemas (`schemas/pulse/v1/calibration.capnp`).

### Who consumes it?

- The compiler uses calibration data to adjust pulses, offsets, and routing.
- The runtime controller applies calibration during execution.
- Device classes may query calibration for setup validation.

### What invariants it carries?

- Calibration data must be consistent with device setup.
- Conflicting calibrations for the same physical channel are rejected.
- Calibration parameters must be within hardware limits.

### How to add new calibration fields?

1. Define new calibration data classes in `laboneq.dsl.calibration`.
2. Extend the payload builder to merge and convert new calibration fields.
3. Update Cap'n Proto schemas and Rust serialization if needed.
4. Add compiler and runtime support for the new calibration data.
5. Add tests covering calibration merging and application.

---

## Artifact types

### What exists?

Artifacts are the outputs of compilation used by the runtime controller and devices. They include:

- SeqC source code and ELF binaries
- Sampled waveform pools and wave indices
- Command tables and phase-increment maps
- Integration weights and kernels
- Pulse-to-waveform provenance maps
- Result handle maps and metadata

Artifacts are encapsulated in `ArtifactsCodegen` within the `ScheduledExperiment` model (`src/python/laboneq/data/scheduled_experiment.py`).

Rationale

Artifacts represent the concrete instructions and data uploaded to hardware devices for execution. They bridge the gap between abstract experiment definitions and device-specific control.

Source Location

- Code generation wrappers: `src/python/laboneq/compiler/seqc/code_generator.py`
- Rust code generator: `src/rust/laboneq-rust/src/lib.rs`
- Scheduled experiment data model: `src/python/laboneq/data/scheduled_experiment.py`

### Who consumes it?

- The runtime controller uploads artifacts to devices.
- Device classes interpret artifacts for hardware programming.
- Developers extending code generation add new artifact types here.

### What invariants it carries?

- Artifacts must be consistent with the scheduled IR and device setup.
- Waveform magnitudes must respect device constraints (e.g., SHFQA scaling).
- Command tables and integration weights must match hardware expectations.

### How to add new artifact types?

1. Extend the Rust code generator to produce new artifact data.
2. Update Python wrappers to expose new artifact fields.
3. Modify device classes to consume and upload new artifacts.
4. Add serialization and storage support if needed.
5. Add tests verifying artifact correctness and runtime usage.

---

## Tests and testing

### What exists?

The repository contains extensive tests covering:

- DSL correctness and usage (`src/python/laboneq/testing/experiments/`)
- Compiler workflow and passes (`src/python/laboneq/compiler/workflow/`)
- Scheduler and IR validation (Rust tests in `src/rust/laboneq-scheduler/tests/`)
- Device setup and runtime controller (`src/python/laboneq/controller/tests/`)
- Calibration merging and payload building (`src/python/laboneq/implementation/payload_builder/tests/`)
- Integration and end-to-end compilation tests

Rationale

Testing ensures correctness, prevents regressions, and validates that extensions integrate properly.

Source Location

- Python tests: `src/python/laboneq/testing/`
- Rust tests: `src/rust/laboneq-scheduler/tests/`, `src/rust/laboneq-ir/tests/`
- Example notebooks: `examples/` directory

### Who consumes it?

- Developers extending any part of the system.
- Continuous integration pipelines.

### What invariants it carries?

- Tests must cover new features and edge cases.
- Tests should verify invariants such as timing constraints, device compatibility, and calibration consistency.

### How to add new tests?

1. Add unit tests for new DSL operations or compiler passes.
2. Add integration tests compiling and running example experiments.
3. Add device-specific tests for new hardware support.
4. Use existing test helpers and fixtures for consistency.
5. Run tests locally and in CI before merging.

---

## Compatibility shims

### What exists?

Compatibility shims handle backward compatibility and cross-version support, especially for:

- Compiler settings and options (`src/python/laboneq/compiler/workflow/compat.py`)
- Device setup compatibility layers
- Parameter conversion and normalization
- Legacy serialization formats

Rationale

To maintain stable user APIs and experiment reproducibility across LabOne Q versions.

Source Location

- Compatibility bridge: `src/python/laboneq/compiler/workflow/compat.py`
- Legacy adapters: `src/python/laboneq/implementation/legacy_adapters/`
- Serialization helpers: `src/python/laboneq/serializers/_legacy/`

### Who consumes it?

- Compiler workflow during experiment lowering.
- Runtime controller when loading legacy experiments.
- Developers maintaining backward compatibility.

### What invariants it carries?

- Shims must preserve experiment semantics.
- Deprecated features should be clearly marked.
- Compatibility code should be isolated to minimize technical debt.

### How to add new compatibility shims?

1. Identify the API or data format change requiring compatibility.
2. Implement conversion or normalization logic in `compat.py` or legacy adapters.
3. Add tests verifying backward compatibility.
4. Document deprecation and migration paths.

---

## Debugging and maintenance checklist

Maintaining LabOne Q requires attention to several key invariants and debugging strategies:

| Area | Checklist Item | Notes |
|-------|----------------|-------|
| DSL | Validate DSL tree structure and unique UIDs | Use `Section` and `Experiment` helpers to inspect trees |
| Payload building | Check calibration merges and signal conflicts | Use `ExperimentInfoBuilder` diagnostics |
| Compiler | Verify IR timing and scheduling constraints | Use Rust IR validation passes and Python compiler reports |
| Scheduler | Confirm sampling-rate commensurability and loop unrolling | Review `laboneq-scheduler` logs and errors |
| Code generation | Validate artifact consistency and waveform scaling | Use code generator unit tests and runtime logs |
| Runtime controller | Confirm device setup fingerprint matches compiled experiment | Use `Controller` validation methods |
| Device support | Check device-specific recipe validation and emulation flags | Use device class unit tests |
| Compatibility | Test legacy experiment loading and option normalization | Use compatibility test suites |
| Tests | Ensure coverage for new features and edge cases | Run full test suite before commits |

### Debugging tips

- Enable detailed logging in Rust compiler via `init_logging()` (`src/rust/laboneq-compiler-py/src/lib.rs`).
- Use pulse sheet viewer (`laboneq.pulse_sheet_viewer`) to visualize compiled experiments.
- Use the `ScheduledExperiment` model to inspect compilation outputs.
- Leverage near-time execution logs and callbacks for runtime debugging.
- Consult device manuals (PQSC, SHFQC, SHFSG) for hardware-specific timing and capability constraints.

---

## Mermaid diagram: Extension points overview

```mermaid
graph TD
  DSL[Python DSL Operations]
  Payload[Payload Builder]
  Compiler[Compiler Passes (Rust)]
  Scheduler[Scheduler & Lowering Passes]
  CodeGen[Code Generation Artifacts]
  Runtime[Runtime Controller & Execution]
  Devices[Device Support & Calibration]
  Compatibility[Compatibility Shims]
  Tests[Tests & Validation]

  DSL --> Payload
  Payload --> Compiler
  Compiler --> Scheduler
  Scheduler --> CodeGen
  CodeGen --> Runtime
  Runtime --> Devices
  Devices --> Compatibility
  Compatibility --> Compiler
  Tests --> DSL
  Tests --> Compiler
  Tests --> Runtime
  Tests --> Devices
```

---

## Summary

Extending and maintaining LabOne Q involves working across multiple layers of abstraction, from the Python DSL frontend through Rust compiler passes to runtime device control. This guide maps the key extension points, their roles, locations, consumers, and invariants. Developers should carefully follow the established patterns for adding new operations, passes, devices, calibration fields, artifacts, and compatibility shims, while ensuring comprehensive testing and adherence to timing and hardware constraints.

---

## References used on this page

1. LabOne Q repository, https://github.com/zhinst/laboneq  
2. Python DSL Experiment, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py  
3. Python DSL Section, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py  
4. ExperimentInfoBuilder, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py  
5. Compiler workflow, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py  
6. Compiler compatibility bridge, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py  
7. Realtime compiler, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/realtime_compiler.py  
8. Rust DSL operation variants, https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs  
9. Rust IR crate, https://github.com/zhinst/laboneq/tree/main/src/rust/laboneq-ir  
10. Rust lowering pass, https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs  
11. Rust QCCS backend preprocessing, https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs  
12. Code generation boundary and outputs, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/seqc/code_generator.py  
13. ScheduledExperiment model, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py  
14. Controller, https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py  
15. Runtime/frontend notes, internal research notes  
16. LabOne Q user manual, https://docs.zhinst.com/labone_q_user_manual/  
17. PQSC device manual, https://docs.zhinst.com/pqsc_user_manual/functional_overview.html  
18. SHFQC device manual, https://docs.zhinst.com/shfqc_user_manual/functional_overview.html  
19. SHFSG device manual, https://docs.zhinst.com/shfsg_user_manual/functional_overview.html
