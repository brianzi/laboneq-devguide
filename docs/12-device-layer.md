# Device communication layer

This chapter provides a detailed developer-oriented overview of the device communication layer in the LabOne Q codebase (`zhinst/laboneq`). It explains the abstractions and mechanisms that enable LabOne Q to interact with Zurich Instruments hardware and the LabOne data server, focusing on the `DeviceCollection` and `DeviceZI` abstractions, data server connections, node writes, upload/ready/done phases, emulation support, setup fingerprinting, and hardware-specific hooks. The roles of the `zhinst.core`, `zhinst.comms`, and `zhinst.toolkit` Python packages in this layer are also discussed.

---

## Maintainer orientation

This page is intended for developers who want to understand or extend the device communication layer of LabOne Q. It assumes familiarity with the overall LabOne Q architecture, the Python DSL frontend, and the compilation and runtime layers described in preceding chapters. The focus here is on the runtime/controller side, specifically the abstractions and implementations that manage device connections, configuration, and execution orchestration.

The page is structured to first introduce the key abstractions (`DeviceCollection`, `DeviceZI`), then describe the interaction with the LabOne data server and the Zurich Instruments communication stack, followed by the phases of experiment upload and execution readiness, emulation support, and finally hardware-specific hooks and setup fingerprinting. Code and file references are provided for direct navigation to the source.

---

## 1. Overview of the device communication layer

LabOne Q is designed to control complex quantum experiments on Zurich Instruments QCCS hardware and compatible third-party devices. The device communication layer is responsible for managing the connections to physical instruments, orchestrating the upload and execution of compiled experiments, and handling runtime interactions such as node writes and result collection.

This layer abstracts the hardware topology and device-specific details behind a common interface, enabling the controller to coordinate multi-device experiments seamlessly. It relies on the Zurich Instruments Python communication stack, primarily the `zhinst.core` API, supplemented by `zhinst.comms` and `zhinst.toolkit` packages, to communicate with the LabOne data server and instruments.

---

## 2. Key abstractions: `DeviceCollection` and `DeviceZI`

### 2.1 `DeviceCollection`

The `DeviceCollection` class is the central container managing all devices involved in an experiment. It encapsulates the set of connected instruments, their logical and physical relationships, and provides bulk operations for setup validation, recipe validation, artifact preparation, execution setup, and result handling.

- **Location:** `src/python/laboneq/controller/devices/device_collection.py`  
- **Role:** Aggregates device instances, manages device lifecycle and orchestration  
- **Consumers:** The `Controller` class uses `DeviceCollection` to delegate device-specific tasks during experiment execution.

The `DeviceCollection` maintains invariants such as consistent device setup fingerprints and synchronized device states. It exposes methods to validate the scheduled experiment against the current hardware configuration, prepare device-specific artifacts, and coordinate execution phases.

### 2.2 `DeviceZI` and device subclasses

`DeviceZI` is an abstract base class representing a Zurich Instruments device. It encapsulates device identity, connection state, trigger-chain topology, and provides abstract hooks for:

- Setup validation  
- AWG configuration discovery  
- Recipe validation  
- Artifact preparation and upload  
- Execution setup and teardown  
- Ready/done waiting  
- Result reading  
- Output disabling  
- Warning and error collection

Concrete subclasses implement these hooks for specific device types such as SHFQA, HDAWG, UHFQA, SHFSG, PQSC, and QHub. These classes use the `zhinst.core` API and related packages to perform device-specific operations.

- **Location:** `src/python/laboneq/controller/devices/device_zi.py` and device-specific files like `device_shfqa.py`, `device_hdawg.py`, etc.  
- **Role:** Abstract device interface and concrete implementations for Zurich Instruments hardware  
- **Consumers:** `DeviceCollection` and `Controller` use `DeviceZI` instances to perform device-level operations.

---

## 3. Interaction with the LabOne data server and Zurich Instruments communication stack

LabOne Q relies on the Zurich Instruments Python communication stack to interact with the LabOne data server and instruments. This stack comprises:

| Package          | Role                                                                                  | Usage in LabOne Q                         |
|------------------|---------------------------------------------------------------------------------------|------------------------------------------|
| `zhinst.comms`   | Bundles the protocol stack for server communication; not intended for direct use      | Used internally by `zhinst.core` and LabOne Q for data server communication |
| `zhinst.core`    | Native Python API for LabOne; provides device and server connection management        | Primary API for device communication and control in LabOne Q controller layer |
| `zhinst.toolkit` | High-level driver package built on `zhinst.core`; provides pythonic device abstractions | Used by LabOne Q controller for runtime/session/instrument access |

The `DeviceZI` classes use `zhinst.core` to establish sessions with the LabOne data server, upload compiled programs, configure device nodes, and monitor execution status. The `zhinst.toolkit` package is leveraged for higher-level device abstractions and utilities, improving code clarity and maintainability.

---

## 4. Node writes and experiment upload phases: upload, ready, done

The device communication layer manages the lifecycle of experiment execution through distinct phases, coordinating node writes and device state transitions.

### 4.1 Node writes

Node writes are the fundamental operations that set device parameters or upload data to the LabOne data server nodes corresponding to instrument settings. These writes are typically batched and synchronized to ensure consistent device configuration.

The `DeviceZI` subclasses provide methods to perform node writes for:

- Uploading waveform and command table data  
- Configuring AWG and sequencer settings  
- Setting trigger and synchronization nodes  
- Applying calibration and mixer settings

### 4.2 Upload phase

During the upload phase, the compiled experiment artifacts (waveforms, command tables, sequencer programs) are transferred to the devices via the LabOne data server. This phase involves:

- Validating the device setup and compiled recipe  
- Preparing device-specific upload data structures  
- Performing node writes to upload waveforms and programs  
- Ensuring all devices have received their respective data

### 4.3 Ready phase

After upload, devices enter the ready phase, where they prepare to execute the experiment. This includes:

- Arming sequencers and AWGs  
- Configuring triggers and synchronization signals  
- Initializing oscillators and local oscillators  
- Clearing previous state and ensuring devices are idle and ready

The device communication layer provides hooks to wait for devices to signal readiness, ensuring synchronized start conditions.

### 4.4 Done phase

Upon experiment completion, the done phase handles:

- Waiting for devices to signal experiment end  
- Reading back results and status nodes  
- Disabling outputs and resetting device state as needed  
- Collecting warnings and errors reported by devices

The `DeviceZI` classes implement methods to perform these tasks, allowing the controller to manage experiment execution cleanly.

---

## 5. Emulation support

LabOne Q supports emulation mode, allowing developers to simulate device behavior without physical hardware. This is essential for testing, debugging, and development in environments without connected instruments.

- **Location:** `src/python/laboneq/controller/devices/zi_emulator.py`  
- **Role:** Provides a mock device implementation that mimics `DeviceZI` behavior without actual hardware communication  
- **Usage:** The controller can instantiate emulated devices to run experiments in software-only mode.

Emulated devices implement the same interface as real devices but simulate node writes, state transitions, and result generation. This enables the controller and runtime layers to operate transparently in emulation mode.

---

## 6. Setup fingerprinting and hardware-specific hooks

### 6.1 Setup fingerprint

The setup fingerprint is a hash or unique identifier representing the current device setup configuration. It is used to:

- Validate that the compiled experiment matches the connected hardware  
- Detect changes in device setup that require recompilation or reconfiguration  
- Ensure consistency between the experiment definition and runtime environment

The fingerprint is computed from the device setup data structures and stored alongside the compiled experiment.

### 6.2 Hardware-specific hooks

Each `DeviceZI` subclass implements hardware-specific hooks to handle device peculiarities, such as:

- Device-specific calibration application  
- AWG and sequencer configuration nuances  
- Trigger and synchronization signal handling  
- Device-specific result reading and processing

These hooks are invoked by the controller during setup validation, artifact preparation, execution setup, and result collection phases.

---

## 7. Summary table of key device communication components

| Component               | Location                                               | Role                                                                                  | Consumers                    |
|-------------------------|--------------------------------------------------------|---------------------------------------------------------------------------------------|-----------------------------|
| `DeviceCollection`      | `src/python/laboneq/controller/devices/device_collection.py` | Manages all devices, orchestrates bulk operations and device lifecycle                 | `Controller`                |
| `DeviceZI` (base class) | `src/python/laboneq/controller/devices/device_zi.py`  | Abstract device interface, defines hooks for setup, upload, execution, and results     | `DeviceCollection`, `Controller` |
| Device subclasses       | `src/python/laboneq/controller/devices/device_*.py`   | Concrete implementations for specific Zurich Instruments devices                      | `DeviceCollection`, `Controller` |
| Emulated device         | `src/python/laboneq/controller/devices/zi_emulator.py`| Mock device for emulation mode                                                        | `Controller`                |
| LabOne data server API  | `zhinst.core` package (external)                       | Provides Python API for LabOne data server and device communication                    | `DeviceZI` subclasses       |
| High-level driver       | `zhinst.toolkit` package (external)                    | Provides pythonic device abstractions and utilities                                  | `DeviceZI` subclasses       |

---

## 8. Mermaid diagram: Device communication layer architecture

```mermaid
graph TD
    Controller --> DeviceCollection
    DeviceCollection --> DeviceZI
    DeviceZI -->|Uses| zhinst_core[zhinst.core API]
    DeviceZI -->|Uses| zhinst_toolkit[zhinst.toolkit]
    DeviceZI -->|Uses| zhinst_comms[zhinst.comms (internal)]
    DeviceZI -->|Implements| DeviceSubclasses[Device Subclasses (SHFQA, HDAWG, PQSC, ...)]
    DeviceZI -->|Implements| ZiEmulator[Emulated Device]
    DeviceCollection -->|Manages| DeviceSubclasses
    DeviceCollection -->|Manages| ZiEmulator
```

---

## 9. Practical developer orientation

### 9.1 Component summary

- A `DeviceCollection` class that aggregates all devices and manages their lifecycle and orchestration.  
- An abstract `DeviceZI` base class defining the device interface and lifecycle hooks.  
- Concrete device classes implementing hardware-specific logic for Zurich Instruments devices.  
- Emulation support via a mock device class.  
- Integration with the LabOne data server through the `zhinst.core` API and higher-level `zhinst.toolkit`.  
- Mechanisms for node writes, experiment upload, ready/done phases, and result collection.  
- Setup fingerprinting to ensure experiment and hardware consistency.

### Design rationale

The device communication layer abstracts the complexity of managing multiple heterogeneous instruments, synchronizing their configuration and execution, and interfacing with the LabOne data server. It enables LabOne Q to provide a unified, high-level experiment control interface while handling the low-level details of device communication and orchestration.

### Source references

- Python source files under `src/python/laboneq/controller/devices/` contain the device communication implementations.  
- The emulation device is in `zi_emulator.py`.  
- The controller and device collection are in `controller.py` and `device_collection.py`.  
- External dependencies on `zhinst.core`, `zhinst.comms`, and `zhinst.toolkit` provide the communication stack.

### 9.4 Integration points

- The `Controller` class consumes `DeviceCollection` and `DeviceZI` instances to manage experiment execution.  
- The runtime layers use device hooks to upload compiled artifacts, configure devices, and collect results.  
- Higher-level application code and user-facing APIs indirectly rely on this layer for hardware interaction.

### 9.5 Invariants

- Device setup fingerprint must match the compiled experiment to avoid runtime errors.  
- Node writes must be synchronized and consistent to ensure correct device configuration.  
- Devices must signal readiness before experiment start and completion after execution.  
- Emulated devices must faithfully mimic device interfaces for testing.  
- Hardware-specific hooks must correctly implement device peculiarities to avoid misconfiguration.

---

## 10. Source code references

| Component           | Source file path                                                                                      | GitHub link                                                                                                                        |
|---------------------|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| `DeviceCollection`  | `src/python/laboneq/controller/devices/device_collection.py`                                        | [device_collection.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_collection.py)   |
| `DeviceZI` base     | `src/python/laboneq/controller/devices/device_zi.py`                                                | [device_zi.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_zi.py)                     |
| SHFQA device        | `src/python/laboneq/controller/devices/device_shfqa.py`                                            | [device_shfqa.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_shfqa.py)               |
| HDAWG device        | `src/python/laboneq/controller/devices/device_hdawg.py`                                            | [device_hdawg.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_hdawg.py)               |
| PQSC device         | `src/python/laboneq/controller/devices/device_pqsc.py`                                             | [device_pqsc.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_pqsc.py)                 |
| Emulated device     | `src/python/laboneq/controller/devices/zi_emulator.py`                                             | [zi_emulator.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/zi_emulator.py)                 |
| Controller          | `src/python/laboneq/controller/controller.py`                                                      | [controller.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py)                           |
| Toolkit adapter     | `src/python/laboneq/controller/toolkit_adapter.py`                                                 | [toolkit_adapter.py](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/toolkit_adapter.py)                 |

---

## 11. Additional notes on emulation and hardware hooks

- The emulated device class (`zi_emulator.py`) implements the same interface as `DeviceZI` but overrides communication methods to simulate device behavior without network or hardware access. This allows for offline testing and development.  
- Hardware-specific hooks in device subclasses handle device-specific quirks such as waveform scaling, trigger routing, and calibration application. These hooks are critical for correct experiment execution and are invoked at various points in the upload and execution lifecycle.  
- The device setup fingerprint is computed during compilation and stored in the `ScheduledExperiment` object. The controller compares this fingerprint with the current device setup to detect mismatches and prevent execution errors.

---

## 12. Summary

The device communication layer in LabOne Q is a sophisticated abstraction that manages the complex interactions between the compiled experiment, the LabOne data server, and the physical Zurich Instruments devices. It provides a clean interface for the controller to orchestrate experiment upload, readiness, execution, and result collection, while encapsulating device-specific details and ensuring consistency through setup fingerprinting. Emulation support enables robust development and testing without hardware. This layer relies heavily on the Zurich Instruments Python communication stack (`zhinst.core`, `zhinst.comms`, and `zhinst.toolkit`) and is implemented primarily in the `src/python/laboneq/controller/devices/` package.

---

## References used on this page

1. LabOne Q repository, `DeviceCollection` source: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_collection.py  
2. LabOne Q repository, `DeviceZI` base class: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_zi.py  
3. LabOne Q repository, device subclasses (e.g., SHFQA): https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/device_shfqa.py  
4. LabOne Q repository, emulated device: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/devices/zi_emulator.py  
5. LabOne Q repository, controller: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/controller.py  
6. LabOne Q repository, toolkit adapter: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/controller/toolkit_adapter.py  
7. Zurich Instruments Python communication stack:  
   - `zhinst.core` PyPI: https://pypi.org/project/zhinst-core/  
   - `zhinst.comms` PyPI: https://pypi.org/project/zhinst-comms/  
   - `zhinst.toolkit` GitHub: https://github.com/zhinst/zhinst-toolkit  
8. LabOne Q user manual, architecture overview: https://docs.zhinst.com/labone_q_user_manual/  
9. LabOne Q Core manual, device setup and session concepts: https://docs.zhinst.com/labone_q_user_manual/core/index.html
