# Ecosystem and Hardware Context

LabOne Q sits above lower-level Zurich Instruments APIs and alongside example/application repositories. The ecosystem matters because the compiler's abstractions are shaped by QCCS device capabilities: sequencer cores, waveform memory, QA/readout channels, physical output grouping, oscillators, and synchronization hardware.

## Related repositories and packages

| Project/package | Role in the ecosystem |
| --- | --- |
| `zhinst/laboneq` | Pulse-level experiment DSL, compiler, code generation, and runtime controller. |
| `zhinst/laboneq-applications` | Higher-level calibration and application workflows built on LabOne Q. |
| `zhinst/zhinst-toolkit` | Pythonic device toolkit layer over lower-level Zurich Instruments APIs. |
| `zhinst/labone-api-examples` | Examples for lower-level LabOne API usage and direct device programming. |
| `zhinst-comms` | Communication infrastructure used below higher-level toolkit/API layers. |

## Device constraints relevant to compilation

The HDAWG, SHFSG, SHFQC, and PQSC families contribute different constraints. HDAWG channel grouping and sequencer ownership influence how output channels are grouped into AWG-local programs.[1] SHFSG/SHFQC output and oscillator structure influence which logical signals can share a resource and how oscillator changes are represented.[2] LabOne Q calibration properties expose some of these constraints at the logical-signal calibration level, including shared local oscillators and pair-wise HDAWG precompensation behavior.[3]

The PQSC provides synchronization context for multi-device QCCS systems. That synchronization context is one reason LabOne Q must preserve a global schedule before fanout into device-local artifacts.

## Hardware-informed compiler design

The compiler's structure reflects the hardware stack. A purely logical compiler would schedule each signal independently and emit independent waveforms. LabOne Q cannot do that because AWG cores, output groups, readout channels, oscillator resources, and command-table capabilities are shared in device-specific ways. The separation between global scheduling and AWG-local lowering is therefore not accidental; it mirrors the separation between experiment timing and sequencer-resource execution.

## References

[1]: https://docs.zhinst.com/hdawg_user_manual/functional_description/specific/awg.html "Zurich Instruments HDAWG User Manual: AWG Sequencer"
[2]: https://docs.zhinst.com/shfsg_user_manual/functional_description/output.html "Zurich Instruments SHFSG User Manual: Output"
[3]: https://docs.zhinst.com/labone_q_user_manual/core/functionality_and_concepts/02_logical_signals/concepts/02_calibration_properties.html "LabOne Q User Manual: Calibration Properties"
