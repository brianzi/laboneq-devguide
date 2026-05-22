# Ecosystem and hardware context

This page provides a comprehensive overview of the LabOne Q software ecosystem and its hardware context within the Zurich Instruments quantum control and measurement platform. It situates the `zhinst/laboneq` codebase in relation to the Zurich Instruments Control and Control System (QCCS), the key hardware components PQSC, SHFQC, and SHFSG, and the related software repositories such as `zhinst-toolkit`, `zhinst-comms`, `labone-api-examples`, and the LabOne Q Applications library. The goal is to orient developers to the roles and interactions of these components, the rationale behind their existence, their locations within the codebase or external repositories, their consumers, and the invariants they maintain.

---

## Maintainer orientation

This page is structured to provide a layered understanding of the LabOne Q ecosystem. It begins with the high-level architectural context of LabOne Q within the Zurich Instruments QCCS hardware and software stack. It then details the roles of the principal hardware devices supported by LabOne Q, focusing on their functional responsibilities and how LabOne Q targets them. Subsequently, the page discusses the software dependencies and related repositories that LabOne Q integrates with or builds upon, clarifying their purposes and relationships.

Maintainers should use this page as a reference to understand where specific functionality or abstractions reside, how LabOne Q fits into the broader Zurich Instruments ecosystem, and how the hardware and software components interoperate. Codebase references include key source file paths and GitHub links for direct inspection. The page also includes a Mermaid diagram illustrating the layered ecosystem architecture, aiding visual comprehension.

---

## 1. LabOne Q in the Zurich Instruments ecosystem

LabOne Q is a quantum experiment programming framework designed to operate within the Zurich Instruments QCCS platform. It provides a Python-based domain-specific language (DSL) for defining quantum experiments independently of the underlying hardware setup, a compiler that transforms these high-level experiment definitions into device-specific real-time programs, and a controller that manages execution and data acquisition on the hardware.

### Architectural placement

The LabOne Q architecture, as depicted in the official README diagram ([LabOne Q README](https://github.com/zhinst/laboneq/blob/main/README.md)) and the LabOne Q user manual ([LabOne Q user manual](https://docs.zhinst.com/labone_q_user_manual/)), positions LabOne Q as a middleware layer between user software and the QCCS hardware. The major components include:

- **DSL**: The Python-based domain-specific language for experiment definition.
- **Session**: Manages connections to the LabOne data server and hardware.
- **Compiler**: Transforms DSL experiments into compiled, device-specific programs.
- **Controller**: Executes compiled experiments on hardware and collects results.
- **Toolkit**: Provides runtime device access and session management.

These components operate on shared data objects such as `DeviceSetup`, `Calibration`, `Experiment`, `CompiledExperiment`, and `Results`. The flow of data and control is from user-defined DSL experiments through compilation and execution, with the session and toolkit managing hardware communication.

> **Definition:**  
> *DeviceSetup* represents the instruments, their options, wiring, and logical signal lines.  
> *Session* represents an active connection to the LabOne data server and connected control system, capable of compiling and executing experiments.  
> *Experiment* is a setup-independent description of the pulse sequence and dynamic process.  
> *Results* store measurement data and metadata needed to reproduce the experiment.

This architecture supports sample-precise RF pulse generation, optimized scheduling, waveform replacement, feedback, dynamic pulses, and memory-efficient code generation on QCCS hardware.

---

## 2. Zurich Instruments QCCS hardware components

LabOne Q targets the Zurich Instruments Control and Control System (QCCS), a modular quantum control platform comprising several specialized instruments. The key devices relevant to LabOne Q are:

| Device | Role | Description | Reference Manual |
|--------|------|-------------|------------------|
| **PQSC** (Programmable Quantum System Controller) | Synchronization and central control | The PQSC acts as the synchronization hub in QCCS, providing 18 ZSync ports to connect downstream devices such as SHFQC, SHFSG, HDAWG, SHFQA, and UHFQA. It distributes the system clock, synchronizes instruments with sub-nanosecond precision, and provides a low-latency data interface for qubit readout results and triggers. | [PQSC manual](https://docs.zhinst.com/pqsc_user_manual/functional_overview.html) |
| **SHFQC** (Superconducting High-Frequency Quantum Controller) | Combined readout and control | The SHFQC+ integrates one Quantum Analyzer readout channel with six Signal Generator control channels. It supports signal generation, qubit measurement, multistate discrimination, and real-time forwarding of results via digital I/O or ZSync. | [SHFQC manual](https://docs.zhinst.com/shfqc_user_manual/functional_overview.html) |
| **SHFSG** (Superconducting High-Frequency Signal Generator) | Microwave signal generation | The SHFSG+ is a microwave signal generator with multiple output channels, each capable of arbitrary waveform generation, modulation, and advanced pulse sequencing. It supports looping, branching, command tables, trigger control, and digital modulation. | [SHFSG manual](https://docs.zhinst.com/shfsg_user_manual/functional_overview.html) |

### Hardware synchronization and timing

The PQSC's ZSync ports enable deterministic synchronization with sub-nanosecond precision, essential for coordinating complex quantum experiments across multiple devices. This hardware synchronization underpins LabOne Q's compiler and runtime design, which must produce globally synchronized per-device programs and distinguish between near-time software control and real-time device execution.

### Hardware capabilities and LabOne Q mapping

LabOne Q's compiler outputs correspond closely to the hardware capabilities of these devices:

- **SHFSG**: The compiler generates sampled waveforms, command tables, and per-channel sequencer code that map directly to the SHFSG's arbitrary waveform generation and advanced pulse sequencing features.
- **SHFQC**: The compiler targets the SHFQC's combined readout and control channels, producing programs that leverage its quantum analyzer, measurement units, and multistate discrimination.
- **PQSC**: The PQSC is recognized as a synchronization device in the compiler backend, with special handling for trigger chains and timing.

---

## 3. Software dependencies and related repositories

LabOne Q depends on and integrates with several Zurich Instruments software packages and repositories that provide device communication, driver abstractions, and example applications.

### 3.1 `zhinst-comms` and `zhinst-core`

- **`zhinst-comms`**: This package bundles the protocol stack required to communicate with the LabOne data server. It is not intended for direct use but is a foundational dependency for higher-level APIs ([PyPI `zhinst-comms`](https://pypi.org/project/zhinst-comms/)).
- **`zhinst-core`**: The native Python API for LabOne, providing direct communication with Zurich Instruments devices. It is distributed as a binary extension and is the base for higher-level toolkits ([PyPI `zhinst-core`](https://pypi.org/project/zhinst-core/)).

LabOne Q uses these packages for runtime communication with hardware devices rather than implementing the full instrument protocol itself.

### 3.2 `zhinst-toolkit`

The `zhinst-toolkit` is a high-level Python driver package built on top of `zhinst.core`. It provides a pythonic and user-friendly interface for controlling Zurich Instruments devices, abstracting low-level details. It requires LabOne software version 25.04 or later and is recommended to run locally on the experiment machine.

LabOne Q depends on `zhinst-toolkit` as a runtime dependency, particularly for session management, device access, and controller orchestration. It is part of the controller/session/instrument-access layer rather than the pulse-level compiler IR.

- Location: [zhinst-toolkit GitHub](https://github.com/zhinst/zhinst-toolkit)
- Source files: `src/zhinst/toolkit/` ([example source](https://github.com/zhinst/zhinst-toolkit/tree/main/src/zhinst/toolkit))

### 3.3 LabOne API examples (`labone-api-examples`)

This repository contains example scripts demonstrating direct control of Zurich Instruments devices via LabOne APIs in Python and MATLAB. It is organized by instrument and API language and requires the `zhinst` package and a running LabOne data server.

LabOne API examples serve as a contrast to LabOne Q's higher-level DSL and compiler approach, illustrating direct low-level instrument programming.

- Location: [labone-api-examples GitHub](https://github.com/zhinst/labone-api-examples)

### 3.4 LabOne Q Applications (`laboneq-applications`)

The LabOne Q Applications library is an application-layer package built on top of LabOne Q. It provides domain-specific quantum elements (e.g., tunable transmons, TWPAs), common quantum operations (measurements, rotations), and pre-built calibration experiments and analyses (resonator spectroscopy, Rabi oscillations, Ramsey interferometry, DRAG calibration, etc.).

This repository is a primary consumer of the LabOne Q DSL and workflow, demonstrating how domain-level experiment libraries sit above the compiler and runtime core.

- Location: [laboneq-applications GitHub](https://github.com/zhinst/laboneq-applications)

---

## 4. Hardware roles and LabOne Q device abstractions

LabOne Q models hardware devices through a layered abstraction that reflects their roles in the QCCS system. The key device types and their roles are summarized below.

| Device Type | Role in QCCS | LabOne Q Abstraction | Key Source Files |
|-------------|--------------|----------------------|------------------|
| **PQSC** | Central synchronization and trigger distribution | `DevicePQSC` class managing synchronization ports, trigger chains, and timing | `src/python/laboneq/controller/devices/device_pqsc.py` |
| **SHFQC** | Combined readout and control | `DeviceSHFQC` class managing quantum analyzer and signal generator channels | `src/python/laboneq/controller/devices/device_shfqc.py` |
| **SHFSG** | Microwave signal generation | `DeviceSHFSG` class managing multi-channel arbitrary waveform generation and sequencer programming | `src/python/laboneq/controller/devices/device_shfsg.py` |

### PQSC: Synchronization and control

The PQSC is the backbone of QCCS synchronization, distributing clock and triggers with sub-nanosecond precision. LabOne Q's `DevicePQSC` abstraction manages this synchronization, ensuring that all downstream devices operate on a common time base. It handles trigger routing, ZSync port management, and timing constraints.

### SHFQC: Quantum analyzer and signal generation

The SHFQC combines a quantum analyzer readout channel with multiple signal generator channels. LabOne Q's `DeviceSHFQC` abstraction manages the complex interplay of readout pulse generation, signal acquisition, integration kernels, and multistate discrimination. It also programs the advanced sequencer features of the signal generator channels.

### SHFSG: Microwave signal generation

The SHFSG provides multiple microwave output channels with arbitrary waveform generation and advanced sequencing capabilities. LabOne Q's `DeviceSHFSG` abstraction programs the sequencer code, command tables, waveform uploads, and trigger control for these channels.

---

## 5. LabOne Q software components and their roles

The LabOne Q codebase is organized into Python and Rust components that collectively implement the DSL, compiler, runtime, and controller layers.

### 5.1 Python DSL frontend

- Location: `src/python/laboneq/dsl/experiment/experiment.py` and `section.py`  
- Role: Provides user-facing abstractions such as `Experiment` and `Section` for defining quantum experiments in Python.  
- Consumers: Users writing quantum experiments, the payload builder, and compiler frontend.  
- Invariants: DSL objects are setup-independent and represent logical experiment structure.

### 5.2 Payload building and compatibility bridge

- Location: `src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py` and `src/python/laboneq/compiler/workflow/compat.py`  
- Role: Converts the Python DSL and device setup into `ExperimentInfo` structures suitable for Rust compiler consumption.  
- Consumers: Rust compiler bridge, scheduler.  
- Invariants: Ensures consistent mapping of logical signals to physical channels, merges calibration data, and validates experiment constraints.

### 5.3 Rust compiler and scheduler

- Location: `src/rust/laboneq-compiler-py/src/lib.rs`, `src/rust/laboneq-ir`, `src/rust/laboneq-scheduler`  
- Role: Implements the core compilation pipeline, including IR construction, scheduling, lowering, and validation.  
- Consumers: Python compiler workflow, code generator.  
- Invariants: Produces timed IR trees with precise sample offsets, enforces timing and hardware constraints.

### 5.4 Code generation

- Location: `src/python/laboneq/compiler/seqc/code_generator.py` and Rust code generator submodules  
- Role: Translates scheduled IR into device-specific code artifacts such as SeqC source, waveforms, command tables, and integration weights.  
- Consumers: Controller runtime, device classes.  
- Invariants: Generates device-compatible code respecting hardware limits and timing.

### 5.5 Controller and runtime

- Location: `src/python/laboneq/controller/controller.py`, `near_time_runner.py`, `recipe_processor.py`  
- Role: Manages experiment execution, device communication, asynchronous result collection, and near-time control loops.  
- Consumers: User applications, LabOne Q session.  
- Invariants: Validates setup fingerprints, manages device state transitions, orchestrates real-time and near-time execution boundaries.

### 5.6 Device layer abstractions

- Location: `src/python/laboneq/controller/devices/`  
- Role: Abstracts hardware devices (PQSC, SHFQC, SHFSG, etc.) with device-specific hooks for setup validation, artifact preparation, execution control, and result reading.  
- Consumers: Controller runtime, compiler backend.  
- Invariants: Enforces device-specific constraints, manages LabOne API connections, supports emulation and real hardware.

---

## 6. Ecosystem architecture diagram

The following Mermaid diagram illustrates the layered architecture of LabOne Q within the Zurich Instruments ecosystem, showing the relationships between user software, LabOne Q components, related repositories, and hardware devices.

```mermaid
graph TD
    subgraph User Software
        DSL[Python DSL (Experiment, Section)]
        Applications[LabOne Q Applications]
        APIExamples[LabOne API Examples]
    end

    subgraph LabOne Q Core
        DSL --> PayloadBuilder[Payload Builder (ExperimentInfoBuilder)]
        PayloadBuilder --> Compiler[Compiler (Python + Rust)]
        Compiler --> CodeGen[Code Generator (SeqC, Waveforms)]
        CodeGen --> Controller[Controller & Runtime]
        Controller --> DeviceLayer[Device Layer Abstractions]
    end

    subgraph Zurich Instruments Software
        DeviceLayer --> Toolkit[zhinst-toolkit]
        Toolkit --> CoreAPI[zhinst-core]
        CoreAPI --> Comms[zhinst-comms]
    end

    subgraph QCCS Hardware
        PQSC[PQSC (Sync & Control)]
        SHFQC[SHFQC (Readout & Control)]
        SHFSG[SHFSG (Signal Generation)]
        PQSC --> SHFQC
        PQSC --> SHFSG
        DeviceLayer --> PQSC
        DeviceLayer --> SHFQC
        DeviceLayer --> SHFSG
    end

    Applications --> DSL
    APIExamples -.-> CoreAPI
```

---

## 7. Summary table of key repositories and their roles

| Repository | Purpose | Relationship to LabOne Q | Location/Reference |
|------------|---------|-------------------------|--------------------|
| `zhinst/laboneq` | Core LabOne Q codebase: DSL, compiler, runtime, controller | Primary codebase implementing LabOne Q functionality | [GitHub](https://github.com/zhinst/laboneq) |
| `zhinst-toolkit` | High-level Python driver for Zurich Instruments devices | Runtime dependency for device communication and session management | [GitHub](https://github.com/zhinst/zhinst-toolkit) |
| `zhinst-comms` | Protocol stack for LabOne data server communication | Underlying communication layer used by `zhinst-core` and `zhinst-toolkit` | [PyPI](https://pypi.org/project/zhinst-comms/) |
| `zhinst-core` | Native Python API for LabOne | Base API for device communication, used by `zhinst-toolkit` and LabOne Q controller | [PyPI](https://pypi.org/project/zhinst-core/) |
| `labone-api-examples` | Low-level instrument control examples | Illustrates direct device programming contrasting LabOne Q DSL | [GitHub](https://github.com/zhinst/labone-api-examples) |
| `laboneq-applications` | Application-layer quantum experiments and analyses | Consumer of LabOne Q DSL and runtime, provides domain-specific experiments | [GitHub](https://github.com/zhinst/laboneq-applications) |

---

## 8. Practical developer orientation

### Component summary

- A Python DSL for experiment definition (`src/python/laboneq/dsl/experiment/experiment.py`).
- A payload builder that converts DSL and setup into compiler inputs (`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py`).
- A Rust-backed compiler and scheduler pipeline (`src/rust/laboneq-compiler-py`, `src/rust/laboneq-ir`, `src/rust/laboneq-scheduler`).
- A code generator producing device-specific artifacts (`src/python/laboneq/compiler/seqc/code_generator.py`).
- A controller runtime managing execution and device communication (`src/python/laboneq/controller/controller.py`).
- Device abstractions for PQSC, SHFQC, SHFSG, and others (`src/python/laboneq/controller/devices/`).
- Integration with Zurich Instruments communication stacks (`zhinst-core`, `zhinst-comms`, `zhinst-toolkit`).
- Application-layer experiment libraries (`laboneq-applications`).

### Design rationale

LabOne Q exists to provide a high-level, hardware-agnostic quantum experiment programming framework that compiles to optimized, sample-precise real-time programs for QCCS hardware. It abstracts complex hardware details, synchronization, and timing constraints, enabling users to focus on experiment logic rather than low-level device programming.

### Source references

- Core LabOne Q codebase: `https://github.com/zhinst/laboneq`
- Related runtime dependencies: `zhinst-toolkit`, `zhinst-core`, `zhinst-comms` (PyPI and GitHub)
- Example and application repositories: `labone-api-examples`, `laboneq-applications`

### Integration points

- Quantum researchers and engineers defining and running experiments on QCCS hardware.
- Application developers building domain-specific experiment libraries.
- LabOne Q maintainers and developers extending compiler, runtime, or device support.

### Invariants

- Experiments are defined independently of hardware setup.
- Compilation produces globally synchronized, sample-precise real-time programs.
- Device abstractions enforce hardware constraints and capabilities.
- Runtime execution respects timing, synchronization, and feedback requirements.
- Communication layers ensure robust, low-latency interaction with hardware.

---

## References used on this page

1. [LabOne Q repository](https://github.com/zhinst/laboneq) — Core LabOne Q codebase and README.  
2. [LabOne Q user manual](https://docs.zhinst.com/labone_q_user_manual/) — Official documentation describing LabOne Q architecture and usage.  
3. [PQSC manual](https://docs.zhinst.com/pqsc_user_manual/functional_overview.html) — Functional overview of the PQSC device.  
4. [SHFQC manual](https://docs.zhinst.com/shfqc_user_manual/functional_overview.html) — Functional overview of the SHFQC device.  
5. [SHFSG manual](https://docs.zhinst.com/shfsg_user_manual/functional_overview.html) — Functional overview of the SHFSG device.  
6. [zhinst-toolkit GitHub](https://github.com/zhinst/zhinst-toolkit) — High-level Python driver package for Zurich Instruments devices.  
7. [zhinst-comms PyPI](https://pypi.org/project/zhinst-comms/) — Protocol stack for LabOne data server communication.  
8. [zhinst-core PyPI](https://pypi.org/project/zhinst-core/) — Native Python API for LabOne.  
9. [labone-api-examples GitHub](https://github.com/zhinst/labone-api-examples) — Low-level instrument control examples.  
10. [laboneq-applications GitHub](https://github.com/zhinst/laboneq-applications) — Application-layer quantum experiments and analyses.
