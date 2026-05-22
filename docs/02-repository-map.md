# Repository and package map

This page provides a comprehensive developer-oriented map of the `zhinst/laboneq` repository. It explains the repository layout, the Python and Rust packages under `src/python` and `src/rust`, the organization of tests, documentation, schemas, inventories, compatibility shims, and guidance on where developers should start exploring the codebase. The goal is to clarify what abstractions exist, why they exist, where they live, who consumes them, and what invariants they carry. This orientation is essential for maintainers and contributors to navigate the complex multi-language codebase effectively.

---

## How to read this page as a maintainer

This page is structured to first present the overall repository layout and then dive into the main Python and Rust packages, followed by auxiliary directories such as tests, docs, and schemas. Each section explains the purpose of the components, their relationships, and their consumers. Where appropriate, code paths and source links are provided for direct reference. The page also highlights key invariants and design rationales to help maintainers understand the architectural decisions behind the code organization.

---

## 1. Repository layout overview

The `zhinst/laboneq` repository is organized as a multi-language project combining Python and Rust code to implement a high-performance quantum experiment compiler and runtime system. The top-level directory structure is summarized in Table 1.

| Directory/File | Description |
|----------------|-------------|
| `src/python/laboneq` | Main Python package implementing the DSL frontend, compiler orchestration, runtime controller, and supporting utilities. |
| `src/rust` | Collection of Rust crates implementing the core compiler, intermediate representations (IR), scheduler, backend preprocessors, and code generators. |
| `schemas` | Cap'n Proto schema files defining the serialization formats for pulses, device setups, experiments, and operations. |
| `docs` | Documentation files including images and the MkDocs configuration. |
| `examples` | Jupyter notebooks and example scripts demonstrating usage of LabOne Q features. |
| `pyproject.toml` | Python packaging and build configuration, including maturin integration for Rust extensions. |
| `Cargo.toml` and `Cargo.lock` | Rust workspace and dependency management files. |

The repository uses a hybrid build system where Python packages are built with `pyproject.toml` and Rust crates are built with Cargo. The Rust crates are compiled into Python extension modules using PyO3 and maturin, exposing Rust functionality to Python.

---

## 2. Python packages under `src/python/laboneq`

The Python codebase is the primary user-facing layer, providing the DSL for experiment definition, compiler orchestration, runtime controller, and device communication abstractions. The Python packages are organized into multiple subpackages, summarized in Table 2.

| Package | Purpose | Key files/examples |
|---------|---------|--------------------|
| `laboneq.dsl` | The domain-specific language (DSL) frontend for defining quantum experiments. Contains classes like `Experiment` and `Section` that users interact with. | [`src/python/laboneq/dsl/experiment/experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py), [`section.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py) |
| `laboneq.compiler` | Compiler orchestration, scheduling, and code generation wrappers. Interfaces with Rust compiler backend via compatibility bridge. | [`src/python/laboneq/compiler/workflow/compiler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py), [`compat.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py) |
| `laboneq._rust` | Python bindings for Rust extension modules, exposing compiler and code generator functionality. | [`src/python/laboneq/_rust/compiler/__init__.pyi`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/_rust/compiler/__init__.pyi), [`codegenerator/__init__.pyi`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/_rust/codegenerator/__init__.pyi) |
| `laboneq.controller` | Runtime controller managing experiment execution, device communication, asynchronous workers, and result collection. | [`src/python/laboneq/controller/controller.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py), [`near_time_runner.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/near_time_runner.py) |
| `laboneq.data` | Data models for experiment descriptions, device setups, calibration, parameters, and results. | [`src/python/laboneq/data/scheduled_experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py) |
| `laboneq.implementation` | Internal implementation details such as payload builders converting DSL to compiler inputs. | [`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py) |
| `laboneq.executor` | Near-time execution IR and interpreter for software control loops and callbacks. | [`src/python/laboneq/executor/executor.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/executor/executor.py) |
| `laboneq.contrub` | Contributed utilities and example helpers, including pulse plotting and data analysis. | - |
| `laboneq.automation` | Automation workflows, serialization, and web viewer components. | - |
| `laboneq.core` | Core utilities, exceptions, serialization, and type definitions. | - |

The Python DSL frontend is the natural starting point for developers interested in experiment definition. The `Experiment` class in `laboneq.dsl.experiment` is the root container for user-defined experiments. Sections, operations, calibration, and device definitions are organized hierarchically under this package.

The compiler orchestration in `laboneq.compiler.workflow.compiler` manages the transformation from DSL to compiler inputs, invoking the Rust compiler backend via the compatibility bridge in `compat.py`. The real-time compiler wrapper in `realtime_compiler.py` handles scheduling and code generation.

The runtime controller in `laboneq.controller` manages experiment execution, device communication, and asynchronous result collection. It consumes the compiled experiment artifacts produced by the compiler and interfaces with the Zurich Instruments LabOne data server via `zhinst-toolkit` and `zhinst-core` dependencies.

---

## 3. Rust crates under `src/rust`

The Rust codebase implements the core compiler, intermediate representations (IR), scheduler, backend preprocessors, and code generators. It is organized as a Cargo workspace with multiple crates, summarized in Table 3.

| Crate | Purpose | Key files/examples |
|-------|---------|--------------------|
| `laboneq-ir` | Central Rust intermediate representation for scheduled experiments. Defines timed IR nodes, experiment containers, and device setup models. | [`src/rust/laboneq-ir/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/lib.rs), [`ir.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/ir.rs), [`node.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs) |
| `laboneq-compiler-py` | PyO3 bridge exposing Rust compiler functionality to Python. Implements experiment deserialization, preprocessing, scheduling, and validation. | [`src/rust/laboneq-compiler-py/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs) |
| `laboneq-scheduler` | Scheduler implementation that validates timing, resolves parameters, chunks experiments, and lowers DSL trees to timed IR nodes. | [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs) |
| `laboneq-dsl` | Rust DSL operation variants representing normalized experiment trees consumed by the scheduler and lowering passes. | [`src/rust/laboneq-dsl/src/operation/variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs) |
| `laboneq-qccs-backend` | Backend-specific preprocessing for QCCS hardware setups, device validation, and signal mapping. | [`src/rust/laboneq-qccs-backend/src/preprocessor.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs) |
| `laboneq-rust` | Root Rust crate exposing code generation and runtime support. | [`src/rust/laboneq-rust/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs) |
| `codegenerator` and `codegenerator-py` | Code generation crates producing SeqC source, waveform pools, command tables, and integration weights for QCCS devices. | [`src/rust/codegenerator-py`](https://github.com/zhinst/laboneq/tree/main/src/rust/codegenerator-py) |

The Rust IR crate `laboneq-ir` defines the fundamental timed tree representation of experiments as `IrNode` objects with `IrKind` variants representing structural nodes (sections, loops, matches) and leaf operations (play pulse, acquire, delay). The `ExperimentIr` container bundles the root IR node with acquisition types, parameter stores, pulse definitions, and device setup context. This IR is the core input to code generation and runtime execution.

The `laboneq-compiler-py` crate acts as the foreign-function interface (FFI) bridge, exposing Rust compiler internals to Python via PyO3. It deserializes Cap'n Proto experiment data, runs backend preprocessing, scheduling, lowering, and validation, and returns compiled experiment artifacts.

The scheduler crate implements passes that validate timing constraints, resolve parameters, unroll loops, and convert the Rust DSL operation tree into the timed IR. The backend crate implements hardware-specific rules and device inventory management for QCCS instruments.

The code generator crates produce device-specific source code and waveform data from the scheduled IR, which the Python controller then uploads to hardware.

---

## 4. Tests, documentation, and schemas

### Tests

The repository contains tests distributed across Python and Rust codebases. Python tests are located under `src/python/laboneq/testing` and its subdirectories, covering experiments, serialization, and utilities. Rust tests are embedded within each crate under `src/rust/*/tests`. These tests validate compiler passes, IR correctness, scheduling, and runtime behaviors.

### Documentation

Documentation files reside under the `docs` directory, including MkDocs configuration and images such as the LabOne Q architecture flowchart (`docs/images/flowchart_QCCS.png`). The developer guide itself is authored in Markdown under `docs/` and rendered with MkDocs Material.

### Schemas

The `schemas` directory contains Cap'n Proto schema files defining the serialization formats for pulses, device setups, experiments, and operations. These schemas are used to serialize experiment data between Python and Rust components. Key schema files include:

- `pulse/v1/calibration.capnp`
- `pulse/v1/device_setup.capnp`
- `pulse/v1/experiment.capnp`
- `pulse/v1/operation.capnp`
- `pulse/v1/section.capnp`
- `pulse/v1/sweep.capnp`

These schemas ensure a stable and efficient binary interface for experiment data exchange.

---

## 5. Inventories and compatibility shims

### Inventories

The repository maintains inventories of devices, signals, and parameters primarily in the Rust backend preprocessing and Python payload building layers.

- The Rust `laboneq-qccs-backend` crate builds a `QccsBackendPreprocessedData` structure that records AWG devices, signal metadata, channel mappings, and lead delays. It validates device combinations and injects synthetic signals for synchronization.

- The Python `ExperimentInfoBuilder` in `laboneq.implementation.payload_builder` converts the Python DSL and device setup into `ExperimentInfo` data structures, resolving logical to physical signals, merging calibrations, and classifying signals by type.

These inventories are crucial invariants that ensure consistent device and signal mapping throughout compilation and runtime.

### Compatibility shims

The Python compiler workflow includes a compatibility bridge in `laboneq.compiler.workflow.compat` that converts Python `ExperimentInfo` into Rust `Experiment` objects. This bridge handles normalization, defaulting, and serialization into Cap'n Proto bytes for Rust consumption.

The Rust PyO3 bridge exposes functions like `build_experiment_capnp()` and `schedule_experiment()` to Python, abstracting away the serialization and deserialization details.

These shims maintain a clean separation between Python user-facing abstractions and Rust internal IRs, enabling incremental migration and interoperability.

---

## 6. Developer orientation: where to start

For developers new to the codebase, the recommended starting points depend on the area of interest:

| Area | Starting point | Description |
|------|----------------|-------------|
| Python DSL frontend | `laboneq.dsl.experiment.Experiment` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py)) | Defines the user-facing experiment container and section hierarchy. |
| Payload building | `ExperimentInfoBuilder` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py)) | Converts DSL and setup into compiler input structures. |
| Compiler orchestration | `Compiler.run()` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py)) | Manages device class resolution, compatibility bridge, scheduling, and code generation. |
| Rust compiler bridge | `laboneq-compiler-py/src/lib.rs` ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs)) | Exposes Rust compiler internals to Python, handles experiment deserialization and scheduling. |
| Rust IR | `laboneq-ir` crate root ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/lib.rs)) | Defines the timed IR nodes and experiment container. |
| Scheduler | `laboneq-scheduler/src/scheduler.rs` ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs)) | Implements scheduling passes and validation. |
| Runtime controller | `laboneq.controller.Controller` ([source](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py)) | Manages experiment execution, device communication, and result collection. |

---

## 7. Mermaid diagram: Repository layered architecture

```mermaid
graph TD
    subgraph Python Layer
        DSL[DSL Frontend (laboneq.dsl)]
        Payload[Payload Builder (implementation.payload_builder)]
        Compiler[Compiler Orchestration (compiler.workflow)]
        Controller[Runtime Controller (controller)]
        Executor[Near-time Executor (executor)]
        RustBindings[PyO3 Rust Bindings (_rust)]
    end

    subgraph Rust Layer
        IR[Intermediate Representation (laboneq-ir)]
        DSL_Rust[Rust DSL (laboneq-dsl)]
        Scheduler[Scheduler (laboneq-scheduler)]
        Backend[Backend Preprocessor (laboneq-qccs-backend)]
        Codegen[Code Generator (codegenerator)]
        CompilerPy[Compiler Py Bridge (laboneq-compiler-py)]
    end

    DSL --> Payload --> Compiler --> RustBindings --> CompilerPy --> Scheduler --> IR --> Codegen
    Controller --> RustBindings
    Controller --> Executor
    Backend --> Scheduler
    DSL_Rust --> Scheduler
```

---

## 8. Detailed package and directory map

### 8.1 Python `src/python/laboneq`

This directory contains the main Python package with many subpackages. Key subdirectories and their roles:

| Subdirectory | Description |
|--------------|-------------|
| `dsl` | User-facing domain-specific language for experiment definition. Includes experiment, section, pulse, calibration, device, and serialization modules. |
| `compiler` | Compiler orchestration, scheduling wrappers, code generation, and compatibility bridge. |
| `_rust` | Python bindings for Rust compiler and code generator extensions. |
| `controller` | Runtime controller managing experiment execution, device communication, and asynchronous workers. |
| `data` | Data models for experiments, calibration, parameters, and results. |
| `implementation` | Internal implementation details such as payload builders converting DSL to compiler inputs. |
| `executor` | Near-time execution IR and interpreter for software control loops and callbacks. |
| `automation` | Automation workflows, serialization, and web viewer components. |
| `contrib` | Contributed utilities, example helpers, and pulse plotting tools. |
| `core` | Core utilities, exceptions, serialization, and type definitions. |
| `instrumentation` | Instrumentation and diagnostics helpers. |
| `pulse_sheet_viewer` | Pulse sheet visualization tools. |
| `serializers` | Serialization implementations and legacy adapters. |
| `simulator` | Output simulation for compiled experiments. |
| `testing` | Test utilities and experiment verifiers. |
| `workflow` | Experiment workflows, blocks, logbooks, options, and tasks. |

---

### 8.2 Rust crates under `src/rust`

The Rust workspace contains multiple crates, each with a focused responsibility:

| Crate | Directory | Description |
|-------|-----------|-------------|
| `laboneq-ir` | `laboneq-ir` | Defines the core intermediate representation (IR) for scheduled experiments, including timed IR nodes and experiment containers. |
| `laboneq-compiler-py` | `laboneq-compiler-py` | PyO3 bridge exposing Rust compiler internals to Python, handling experiment deserialization, preprocessing, scheduling, and validation. |
| `laboneq-scheduler` | `laboneq-scheduler` | Implements the scheduler passes that validate timing, resolve parameters, chunk experiments, and lower DSL trees to timed IR nodes. |
| `laboneq-dsl` | `laboneq-dsl` | Defines Rust DSL operation variants representing normalized experiment trees consumed by the scheduler and lowering passes. |
| `laboneq-qccs-backend` | `laboneq-qccs-backend` | Backend-specific preprocessing for QCCS hardware setups, device validation, and signal mapping. |
| `laboneq-rust` | `laboneq-rust` | Root Rust crate exposing code generation and runtime support to Python. |
| `codegenerator` | `codegenerator` | Code generation producing SeqC source, waveform pools, command tables, and integration weights for QCCS devices. |
| `codegenerator-py` | `codegenerator-py` | Python bindings for the code generator crate. |
| Other crates | Various | Utility crates for logging, error handling, units, numeric arrays, tracing, and Cap'n Proto schema support. |

---

## 9. Key invariants and abstractions

- **DSL vs IR separation**: The Python DSL (`laboneq.dsl`) provides user-friendly experiment definition abstractions, while the Rust DSL (`laboneq-dsl`) and IR (`laboneq-ir`) represent normalized, scheduled, and timed experiment trees for compilation and code generation.

- **Compatibility bridge**: The Python-to-Rust compatibility bridge (`laboneq.compiler.workflow.compat`) converts Python `ExperimentInfo` into Rust `Experiment` objects serialized via Cap'n Proto, ensuring a clean foreign-function interface.

- **Scheduling and lowering**: The scheduler (`laboneq-scheduler`) validates timing constraints, resolves parameters, chunks experiments, and lowers the Rust DSL tree into timed IR nodes (`IrNode`), which are the canonical representation for code generation.

- **Backend preprocessing**: The QCCS backend (`laboneq-qccs-backend`) enforces hardware-specific constraints, device inventory, and signal mapping before scheduling.

- **Code generation**: The code generator crates produce device-specific source code, waveform data, and command tables consumed by the runtime controller.

- **Runtime controller**: The Python controller (`laboneq.controller`) manages experiment execution, device communication via `zhinst-toolkit` and `zhinst-core`, asynchronous workers, and result collection.

- **Near-time execution**: The near-time executor (`laboneq.executor`) interprets a Python-side execution IR for software-controlled loops and callbacks, separate from the real-time scheduled IR.

---

## 10. Summary table of important source files with links

| Component | Source file | Description |
|-----------|-------------|-------------|
| Python DSL Experiment | [`src/python/laboneq/dsl/experiment/experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py) | User-facing experiment container class. |
| Python DSL Section | [`src/python/laboneq/dsl/experiment/section.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py) | Section class and DSL helpers. |
| ExperimentInfoBuilder | [`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py) | Converts DSL and setup into compiler input structures. |
| Compiler orchestration | [`src/python/laboneq/compiler/workflow/compiler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py) | Compiler entry point and orchestration. |
| Compiler compatibility bridge | [`src/python/laboneq/compiler/workflow/compat.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py) | Python-to-Rust experiment conversion. |
| Realtime compiler wrapper | [`src/python/laboneq/compiler/workflow/realtime_compiler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/realtime_compiler.py) | Scheduling and code generation orchestration. |
| Rust IR crate root | [`src/rust/laboneq-ir/src/lib.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/lib.rs) | IR definitions and builders. |
| Rust IR node model | [`src/rust/laboneq-ir/src/node.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs) | Timed IR node structure. |
| Rust DSL operation variants | [`src/rust/laboneq-dsl/src/operation/variants.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs) | Rust DSL experiment tree nodes. |
| Rust scheduler | [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs) | Scheduler passes and validation. |
| Rust QCCS backend preprocessor | [`src/rust/laboneq-qccs-backend/src/preprocessor.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs) | Hardware-specific device validation and signal mapping. |
| Python runtime controller | [`src/python/laboneq/controller/controller.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py) | Experiment execution and device communication. |
| ScheduledExperiment model | [`src/python/laboneq/data/scheduled_experiment.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py) | Controller-facing compiled experiment data. |

---

## 11. Summary

The `zhinst/laboneq` repository is a complex multi-language project combining Python and Rust to provide a high-level quantum experiment DSL, a performant compiler pipeline, and a runtime controller for Zurich Instruments QCCS hardware. The Python packages under `src/python/laboneq` implement the user-facing DSL, compiler orchestration, runtime controller, and utilities. The Rust crates under `src/rust` implement the core compiler IR, scheduler, backend preprocessing, and code generation. The repository also includes tests, documentation, Cap'n Proto schemas, and example notebooks.

Maintainers should understand the clear separation between user-facing Python DSL abstractions and the internal Rust IR and scheduling layers, bridged by compatibility shims and serialization. The runtime controller consumes compiled experiment artifacts and manages hardware communication asynchronously. This layered architecture supports extensibility, hardware abstraction, and efficient real-time execution.

---

## References used on this page

1. LabOne Q repository: https://github.com/zhinst/laboneq  
2. Python DSL Experiment source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/experiment.py  
3. Python DSL Section source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py  
4. ExperimentInfoBuilder source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py  
5. Compiler workflow source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compiler.py  
6. Compiler compatibility bridge source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/compat.py  
7. Realtime compiler wrapper source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/realtime_compiler.py  
8. Rust IR crate root source: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/lib.rs  
9. Rust IR node model source: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-ir/src/node.rs  
10. Rust DSL operation variants source: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-dsl/src/operation/variants.rs  
11. Rust scheduler source: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs  
12. Rust QCCS backend preprocessor source: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-qccs-backend/src/preprocessor.rs  
13. Python runtime controller source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py  
14. ScheduledExperiment model source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/data/scheduled_experiment.py  
15. LabOne Q user manual: https://docs.zhinst.com/labone_q_user_manual/  
16. zhinst-toolkit repository: https://github.com/zhinst/zhinst-toolkit  
17. zhinst-core PyPI: https://pypi.org/project/zhinst-core/  
18. zhinst-comms PyPI: https://pypi.org/project/zhinst-comms/
