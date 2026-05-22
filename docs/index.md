# LabOne Q Developer Guide

Welcome to the comprehensive developer guide for **LabOne Q**, the quantum experiment programming framework developed by Zurich Instruments under the `zhinst/laboneq` repository. This guide is intended to provide an executive orientation and detailed developer roadmap to the LabOne Q codebase, its architecture, and its role within the broader Zurich Instruments ecosystem. It is designed for developers, maintainers, and contributors who seek a deep understanding of the system beyond the user-facing documentation.

---

## Table of Contents

- [Purpose and Scope](#purpose-and-scope)  
- [Repository Snapshot and Layout](#repository-snapshot-and-layout)  
- [Maintainer orientation](#maintainer-orientation)  
- [High-Level Layered Architecture](#high-level-layered-architecture)  
- [Reading Paths and Documentation Structure](#reading-paths-and-documentation-structure)  
- [References used on this page](#references-used-on-this-page)  

---

## Purpose and Scope

LabOne Q is a Python-based domain-specific language (DSL) and compiler framework designed to enable precise, sample-accurate quantum experiments on Zurich Instruments’ Quantum Control and Classical Control System (QCCS) hardware, as well as third-party control devices. Unlike the public user manual, which targets end users and experiment designers, this developer guide focuses on the internal structure, abstractions, and workflows of the LabOne Q codebase.

The guide explains the following core aspects:

- The **repository layout** and package organization, including Python and Rust components.
- The **Python DSL frontend**, which provides the user-facing experiment definition API.
- The **payload-building bridge**, which converts DSL objects into compiler inputs.
- The **Rust-backed compiler and scheduling pipeline**, including intermediate representations (IRs).
- The **code generation artifacts** and their mapping to hardware instructions.
- The **runtime and controller execution model**, including near-time and real-time orchestration.
- The **Zurich Instruments communication dependencies** and hardware context.
- Related repositories and ecosystem components, such as `zhinst-toolkit` and `laboneq-applications`.

This guide explicitly distinguishes user-facing abstractions (e.g., Python DSL classes) from internal IRs and compatibility shims, providing clarity on the roles and invariants of each layer.

---

## Repository Snapshot and Layout

The LabOne Q repository is hosted at [https://github.com/zhinst/laboneq](https://github.com/zhinst/laboneq). It is a multi-language project combining Python and Rust code, with a layered architecture that reflects the compilation and execution pipeline.

### Top-Level Structure

| Directory / File | Description |
|------------------|-------------|
| `src/python/laboneq` | Main Python package containing DSL, compiler orchestration, controller, runtime, and utilities. |
| `src/python/laboneq/_rust` | Python bindings to Rust crates, exposing compiler and code generator extensions. |
| `src/rust/laboneq-ir` | Rust crate defining the core intermediate representation (IR) for scheduled experiments. |
| `src/rust/laboneq-scheduler` | Rust crate implementing scheduling, timing grids, and lowering passes. |
| `src/rust/laboneq-compiler-py` | Rust crate providing the Python-Rust compiler bridge via PyO3. |
| `src/rust/laboneq-qccs-backend` | Backend-specific preprocessing for QCCS hardware mapping. |
| `src/rust/laboneq-rust` | Root Rust crate exposing core compiler and code generation APIs. |
| `schemas/pulse/v1` | Cap’n Proto schema files defining serialization formats for experiments, devices, pulses, and operations. |
| `examples/` | Jupyter notebooks demonstrating device setup, session usage, DSL experiments, workflows, and serialization. |
| `docs/` | Documentation assets including images and pyproject.toml for MkDocs. |

### Python Package Highlights

- **DSL frontend**: `src/python/laboneq/dsl/experiment/experiment.py` and `section.py` define the main user-facing classes for experiment construction.
- **Payload builder**: `src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py` converts DSL trees into compiler input structures.
- **Compiler workflow**: `src/python/laboneq/compiler/workflow/compiler.py` orchestrates compilation, including device-class resolution and chunking.
- **Scheduler wrapper**: `src/python/laboneq/compiler/scheduler/scheduler.py` calls into Rust scheduling logic.
- **Code generation**: `src/python/laboneq/compiler/seqc/code_generator.py` wraps Rust code generation for SeqC output.
- **Runtime controller**: `src/python/laboneq/controller/controller.py` manages experiment execution and device communication.
- **Near-time execution**: `src/python/laboneq/controller/near_time_runner.py` implements near-time loop execution and callback handling.
- **Executor IR**: `src/python/laboneq/executor/executor.py` defines the near-time execution intermediate representation.
- **Rust extension**: `src/python/laboneq/_rust/compiler/__init__.pyi` and `src/python/laboneq/_rust/codegenerator/__init__.pyi` expose Rust APIs to Python.

### Rust Crate Highlights

- **IR crate**: `src/rust/laboneq-ir/src/ir.rs` and `node.rs` define the timed tree IR (`IrNode`) with operation kinds (`IrKind`).
- **Scheduler**: `src/rust/laboneq-scheduler/src/scheduler.rs` implements scheduling passes, validation, and lowering.
- **Compiler bridge**: `src/rust/laboneq-compiler-py/src/lib.rs` exposes PyO3 bindings for experiment serialization, scheduling, and compilation.
- **QCCS backend**: `src/rust/laboneq-qccs-backend/src/preprocessor.rs` handles hardware-specific setup and device mapping.
- **Code generator**: `src/rust/laboneq-rust/src/lib.rs` provides code generation entry points for SeqC and ELF artifacts.

---

## Maintainer orientation
This page serves as a high-level orientation and roadmap. To effectively use it:

- **Focus on the layered architecture diagram** to understand the major components and their interactions.
- Use the **repository snapshot** to locate source files and packages relevant to your task.
- Follow the **reading paths** to dive deeper into specific subsystems, such as the Python DSL frontend or the Rust compiler core.
- Refer to the **references** for authoritative source links and official documentation.
- Note the explicit distinction between **user-facing abstractions** (e.g., Python DSL classes) and **internal IRs** (Rust `IrNode` trees), which is crucial for understanding data flow and transformation.
- Use the **outline of the documentation site** as a checklist for exploring or contributing to the codebase.

This page is not a substitute for reading source code but a structured guide to navigate it efficiently.

---

## High-Level Layered Architecture

LabOne Q is architected as a layered system bridging user experiment definitions to hardware execution on QCCS instruments. The architecture separates concerns into distinct layers, each with clear responsibilities, data models, and invariants.

```mermaid
graph TD
    subgraph User_Software [User Software]
        UserApplications[User Applications]
        LabOneQDSL[LabOne Q DSL]
    end

    subgraph Python_DSL_Frontend [Python DSL Frontend]
        ExperimentDSL[Experiment DSL]
        SectionDSL[Section DSL]
        OperationDSL[Operation DSL]
    end

    subgraph Payload_Bridge [Payload Builder & Compatibility Bridge]
        ExperimentInfoBuilder[ExperimentInfoBuilder]
        SetupHelper[SetupHelper]
        ExperimentInfo[ExperimentInfo]
        CompilerCompatibilityBridge[Compiler Compatibility Bridge]
    end

    subgraph Rust_Compiler [Rust Compiler & Scheduler]
        RustDSLOperations[Rust DSL Operations]
        SchedulerPasses[Scheduler Passes]
        IRNodes[IR Nodes]
        ExperimentIr[Experiment IR]
    end

    subgraph Code_Generation [Code Generation]
        SeqCCodeGenerator[SeqC Code Generator]
        ELFArtifacts[ELF Artifacts]
        WaveformMaps[Waveform Maps]
        CommandTables[Command Tables]
    end

    subgraph Runtime_Controller [Runtime & Controller]
        ScheduledExperiment[Scheduled Experiment]
        Controller[Controller]
        NearTimeRunner[Near-Time Runner]
        RecipeData[Recipe Data]
    end

    subgraph Device_Layer [Device Communication & Hardware]
        DeviceCollection[Device Collection]
        DeviceZI[Device ZI]
        LabOneDataServer[LabOne Data Server]
        QCCSHardware[QCCS Hardware]
    end

    UserApplications --> LabOneQDSL
    LabOneQDSL --> ExperimentDSL
    ExperimentDSL --> ExperimentInfoBuilder
    ExperimentInfoBuilder --> CompilerCompatibilityBridge
    CompilerCompatibilityBridge --> RustDSLOperations
    RustDSLOperations --> SchedulerPasses
    SchedulerPasses --> IRNodes
    IRNodes --> SeqCCodeGenerator
    SeqCCodeGenerator --> ScheduledExperiment
    ScheduledExperiment --> Controller
    Controller --> DeviceCollection
    DeviceCollection --> LabOneDataServer
    LabOneDataServer --> QCCSHardware
```

### Layer Descriptions

| Layer | Description | Key Components | Invariants |
|-------|-------------|----------------|------------|
| **User Software** | User-level experiment definitions and applications built on LabOne Q. | `laboneq-applications`, custom user scripts. | User code defines experiments using the DSL; no direct hardware interaction. |
| **Python DSL Frontend** | Provides the Python API for defining experiments, sections, and operations. | `laboneq.dsl.experiment.Experiment`, `Section`, `Operation` classes. | DSL objects are immutable or controlled via context managers; maintain experiment structure invariants. |
| **Payload Builder & Compatibility Bridge** | Converts DSL trees and device setup into compiler input structures; bridges Python and Rust. | `ExperimentInfoBuilder`, `SetupHelper`, `build_rs_experiment()`. | Ensures consistent mapping of logical signals to physical channels; merges calibration data; validates chunking and loops. |
| **Rust Compiler & Scheduler** | Core compilation pipeline implemented in Rust; schedules experiment timing and lowers DSL to IR. | `laboneq-ir::IrNode`, `laboneq-scheduler::Scheduler`, `laboneq-compiler-py` bridge. | IR is a timed tree with precise sample offsets; scheduling enforces timing constraints and hardware compatibility. |
| **Code Generation** | Generates device-specific code artifacts such as SeqC programs, waveforms, and command tables. | `laboneq._rust.codegenerator`, `SeqCProgram`, waveform pools. | Generated code is consistent with scheduled IR; waveforms satisfy device constraints (e.g., amplitude limits). |
| **Runtime & Controller** | Manages experiment execution, including near-time loops, real-time boundaries, and result collection. | `ScheduledExperiment`, `Controller`, `NearTimeRunner`, `RecipeData`. | Execution respects setup fingerprints; callbacks and replacements are handled asynchronously; results are collected reliably. |
| **Device Communication & Hardware** | Interfaces with LabOne data server and QCCS instruments; manages device-specific protocols. | `DeviceCollection`, `DeviceZI`, `zhinst.core`, `zhinst.comms`, `zhinst-toolkit`. | Device state is synchronized; setup validation and artifact upload are device-specific; emulation supported. |

---

## Reading Paths and Documentation Structure

The developer documentation is organized into a sequence of pages, each focusing on a subsystem or conceptual layer. Below is the proposed site structure with brief descriptions to guide your reading:

| File | Page Title | Purpose |
|-------|------------|---------|
| `index.md` | LabOne Q Developer Guide | Executive orientation, scope, source snapshot, reading path, and layered architecture. (This page) |
| `01-ecosystem.md` | Ecosystem and Hardware Context | Position LabOne Q within Zurich Instruments ecosystem, including dependencies like `zhinst-toolkit` and related repositories. |
| `02-repository-map.md` | Repository and Package Map | Detailed layout of source directories, Python and Rust packages, tests, schemas, and generated artifacts. |
| `03-dependencies.md` | Dependencies and Build System | Explanation of Python packaging, Rust extension building with maturin, and third-party dependencies rationale. |
| `04-frontend-dsl.md` | Python DSL Frontend | Documentation of user-facing DSL classes such as `Experiment`, `Section`, operations, calibration, and syntactic sugar. |
| `05-payload-and-compiler-input.md` | Payload Building and Compiler Inputs | Description of `ExperimentInfoBuilder`, signal mapping, calibration merging, parameter conversion, and validation. |
| `06-compiler-overview.md` | Compiler Workflow Overview | Top-level orchestration of compilation, including chunking, scheduling, and code generation handoff. |
| `07-ir-models.md` | Intermediate Representations | Taxonomy of IRs from Python DSL tree to Rust DSL tree, scheduled IR, and runtime execution IR. |
| `08-rust-compiler-core.md` | Rust Compiler Core | Deep dive into Rust crates for IR, scheduling, and Python bindings. |
| `09-scheduling-and-timing.md` | Scheduling, Timing Grids, and Section Semantics | Detailed explanation of timing constraints, grid alignment, and section-level scheduling logic. |
| `10-code-generation-artifacts.md` | Code Generation Artifacts | Mapping of scheduled IR to SeqC, ELF, command tables, and waveform pools. |
| `11-runtime-controller.md` | Runtime and Controller Execution | Orchestration of experiment execution, near-time loops, and device interaction. |
| `12-device-layer.md` | Device Communication Layer | Low-level device drivers, LabOne API integration, and hardware abstraction. |
| `13-results-and-data.md` | Results, Handles, and Data Shapes | Data model for experiment results, acquisition handles, and post-processing. |
| `14-extension-points.md` | Extension and Maintenance Guide | Guide for adding new operations, device support, or compiler passes. |
| `15-source-reference.md` | Source Reference Map | Direct mapping of functional components to source file locations. |
| `16-glossary.md` | Glossary | Definition of terms and acronyms used in the codebase. |
| `references.md` | References | Links to upstream repositories, official docs, and related papers. |

---

## References used on this page

- [zhinst/laboneq Repository](https://github.com/zhinst/laboneq)
- [LabOne Q User Manual](https://docs.zhinst.com/labone_q_user_manual/)
- [Zurich Instruments QCCS Documentation](https://docs.zhinst.com/qccs_user_manual/)
