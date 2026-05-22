# Scheduling, timing grids, and section semantics

This page provides a comprehensive developer-oriented explanation of the scheduling and timing mechanisms in the `zhinst/laboneq` codebase. It covers the responsibilities of the scheduler, the semantics of sections and timing grids, acquisition loops, repetition modes, delay and precompensation semantics, pseudo-random number generation (PRNG) and match/case constructs, feedback constraints, and validation rules. The emphasis is on concrete structure, implementation boundaries, integration points, and correctness constraints.

---

## Maintainer orientation

This page is intended for developers working on or extending the LabOne Q compiler and runtime system, especially those involved in scheduling, timing, and real-time execution. It assumes familiarity with the overall LabOne Q architecture and the Python DSL frontend, as well as the Rust compiler backend.

The page is structured to first explain the high-level concepts and abstractions, then delve into the key components and their interactions, referencing relevant source files and modules. Code paths and source links are provided for direct inspection. Inferred design details are identified explicitly when the code is more authoritative than the public documentation.

Developers maintaining or enhancing scheduling should pay particular attention to the distinction between the Python DSL experiment tree, the Rust DSL operation tree, the scheduled IR nodes, and the final IR nodes with concrete timing. Understanding the timing grid propagation and section timing modes is critical for correct scheduling and validation.

---

## 1. Scheduler responsibilities and architecture

### 1.1 Scheduler responsibilities

The scheduler in LabOne Q is responsible for transforming a normalized experiment description into a timed, device-compatible representation suitable for code generation and real-time execution. This involves:

- Validating timing constraints and device compatibility.
- Computing timing grids and aligning operations.
- Resolving repetition modes and loop semantics.
- Handling acquisition loops and averaging.
- Managing delay semantics and precompensation.
- Lowering high-level DSL constructs into timed IR nodes.
- Validating feedback and match/case constraints.
- Producing a scheduled IR tree with concrete sample offsets and durations.

The scheduler thus bridges the gap between the user-facing Python DSL and the low-level device-specific code generation.

### 1.2 Scheduler source references

The scheduler is primarily implemented in Rust within the `laboneq-scheduler` crate:

- Entry point: [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs)
- Lowering passes: [`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs)
- Validation: [`src/rust/laboneq-scheduler/src/analysis/validate_ir.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/analysis/validate_ir.rs)

The Python wrapper around the scheduler is in:

- [`src/python/laboneq/compiler/scheduler/scheduler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/scheduler/scheduler.py)
- [`src/python/laboneq/compiler/workflow/realtime_compiler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/realtime_compiler.py)

### 1.3 Scheduler integration points

- The Python compiler workflow (`Compiler.run`) calls the scheduler to produce a scheduled IR.
- The code generator consumes the scheduled IR to produce device-specific code and artifacts.
- The controller uses the scheduled experiment metadata for runtime orchestration.
- Validation tools and diagnostics use scheduler outputs to verify timing correctness.

---

## 2. Timing grids and alignment

### 2.1 Concept of timing grids

LabOne Q uses discrete timing grids to represent sample-precise timing offsets and durations. Each signal or device may have its own base sample rate, and the scheduler must align operations across signals with potentially different rates.

The scheduler propagates timing grids through the experiment tree, escalating to least-common-multiple (LCM) grids when combining children with different grids. This ensures that all operations can be scheduled on a common sample grid without fractional offsets.

### 2.2 Grid escalation and propagation

The scheduler starts with the system grid, typically the highest sample rate among devices. Child nodes inherit or escalate their grids to ensure commensurability. For example:

- Child nodes with different sample rates escalate to the LCM of their grids.
- Loop constructs may compress grids internally but must align with parent grids.
- Sequencer grids and compressed-loop grids are escalated to maintain timing consistency.

This grid propagation is implemented in the lowering pass [`lower_experiment/mod.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs#L600).

### 2.3 Section alignment

Sections in the Python DSL (`laboneq.dsl.experiment.section.Section`) carry an alignment attribute (`SectionAlignment`) that specifies how the section start aligns with the global timing grid. The scheduler enforces these alignments during lowering.

---

## 3. Section timing modes and semantics

### 3.1 Section timing modes

Sections have a `SectionTimingMode` attribute that controls how their timing is interpreted:

- **Strict timing**: The section length and alignment are strictly enforced; no timing slack is allowed.
- **Flexible timing**: The section can be extended or compressed by the scheduler to accommodate hardware constraints.
- **On system grid**: Sections aligned on the system grid have timing constraints propagated accordingly.

The timing mode affects how the scheduler validates and schedules the section's children.

### 3.2 Section triggers and play-after semantics

Sections may specify triggers that synchronize their start with external events or other sections. The `play_after` attribute allows specifying dependencies between sections, ensuring correct temporal ordering.

The scheduler uses these attributes to build a global schedule that respects inter-section dependencies and triggers.

---

## 4. Acquire loops and repetition modes

### 4.1 Acquire loops

The `AcquireLoopRt` construct in the Python DSL represents the real-time acquisition and averaging loop. It is the boundary between near-time software control and real-time hardware execution.

Key attributes include:

- `acquisition_type`: Defines the type of acquisition (e.g., raw, integrated).
- `averaging_mode`: Specifies how averaging is performed.
- `count`: Number of repetitions.
- `repetition_mode`: Controls timing behavior between repetitions.
- `repetition_time`: Optional fixed repetition time.
- `reset_oscillator_phase`: Whether to reset oscillator phase at loop start.

The scheduler detects the acquire loop and uses it to determine acquisition type and timing constraints.

### 4.2 Repetition modes

Repetition modes control how the scheduler handles timing between loop iterations:

| Mode           | Description                                                                                   |
|----------------|-----------------------------------------------------------------------------------------------|
| Fixed          | Each iteration has a fixed duration equal to the repetition time.                             |
| Continuous     | Iterations run back-to-back without gaps.                                                    |
| Triggered      | Iterations start on external triggers, possibly with variable delays.                        |
| ResetPhase     | Oscillator phase is reset at the start of each iteration.                                    |

The scheduler resolves repetition modes during lowering and applies timing adjustments accordingly.

---

## 5. Delay semantics and precompensation

### 5.1 Delay nodes

Delay operations in the DSL represent explicit wait times on signals. They may carry a `precompensation_clear` flag indicating that precompensation effects should be cleared at that point.

Delays are lowered into timed IR nodes with concrete sample durations.

### 5.2 Precompensation

Precompensation compensates for hardware distortions such as filter responses or cable delays. The scheduler inserts `ClearPrecompensation` nodes as zero-duration markers to reset precompensation state.

Precompensation parameters are validated and clamped in the Python workflow (`precompensation_helpers.py`), ensuring hardware-compatible values.

---

## 6. PRNG and match/case constructs

### 6.1 PRNG setup and loops

Pseudo-random number generation (PRNG) is used for randomized experiments. The DSL includes `PRNGSetup` and `PRNGLoop` constructs that configure and use PRNG sequences.

The scheduler lowers PRNG setup into section nodes carrying PRNG metadata, ensuring deterministic and reproducible random sequences.

### 6.2 Match and case semantics

Match/case constructs implement conditional branching in the real-time experiment. The scheduler resolves near-time match cases, validates feedback constraints, and lowers these constructs into timed IR nodes.

Match nodes carry a target parameter and local feedback flags, while case nodes represent individual branches.

---

## 7. Feedback constraints and validation

### 7.1 Feedback constraints

Feedback operations, such as phase resets and PPC (pump-probe-cancellation) steps, have strict timing and device constraints. The scheduler enforces these constraints, rejecting unsupported combinations or timing violations.

### 7.2 Validation passes

The scheduler includes validation passes that check:

- Timing consistency and grid commensurability.
- Section timing mode compliance.
- Feedback and match/case correctness.
- Acquisition length adjustments.
- Loop unrolling correctness.
- Parameter resolution completeness.

Validation errors are reported early to prevent invalid compiled experiments.

---

## 8. Scheduler data structures and IR nodes

### 8.1 ScheduledNode vs IrNode

The scheduler internally uses `ScheduledNode` objects that carry scheduling constraints and unresolved timing information. After scheduling and lowering, these are converted into `IrNode` objects with concrete sample offsets and durations.

- `ScheduledNode`: Used during scheduling passes, carries constraints.
- `IrNode`: Final timed IR node tree for code generation.

### 8.2 Key IR node kinds

The `IrKind` enum defines node kinds such as:

| Kind                 | Description                                      |
|----------------------|------------------------------------------------|
| Root                 | Root of the experiment IR tree                  |
| Section              | Section node with triggers and timing mode      |
| Loop                 | Loop node with iteration count and kind         |
| PlayPulse            | Pulse play operation on a signal                 |
| Acquire              | Acquisition operation with handle and kernels   |
| Delay                | Delay operation with optional precompensation   |
| ResetOscillatorPhase | Oscillator phase reset operation                  |
| Match                | Conditional match node                            |
| Case                 | Branch case node                                 |
| PrngSetup            | PRNG setup node                                  |
| ClearPrecompensation | Zero-duration node to clear precompensation     |

These nodes form the timed tree representing the scheduled experiment.

---

## 9. Scheduling workflow summary

The scheduling workflow proceeds as follows:

1. **Preprocessing**: Backend-specific setup processing and validation.
2. **Near-time match resolution**: Resolve near-time conditional branches.
3. **Chunking**: Optionally split experiment into chunks for large sweeps.
4. **Lowering**: Convert DSL operation tree into scheduled nodes with timing constraints.
5. **Repetition mode resolution**: Apply repetition mode semantics.
6. **Validation**: Check timing, feedback, and parameter correctness.
7. **Acquisition length adjustment**: Adjust acquisition lengths for hardware constraints.
8. **Loop unrolling**: Unroll loops as needed.
9. **Parameter resolution**: Substitute parameter values.
10. **Timing calculation**: Compute concrete sample offsets and durations.
11. **Conversion to IR nodes**: Produce final timed IR tree.

---

## 10. Practical developer orientation

| Aspect                | Description                                                                                              | Location(s)                                                                                  |
|-----------------------|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Scheduler entry point  | Called by Python `Scheduler` class, invokes Rust scheduler via PyO3 bridge                                | `src/python/laboneq/compiler/scheduler/scheduler.py`<br>`src/rust/laboneq-scheduler/src/scheduler.rs` |
| Timing grids          | Propagated and escalated during lowering to ensure sample alignment                                      | `src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`                                    |
| Section timing modes  | Enforced during lowering and validation, affect scheduling and code generation                            | `src/python/laboneq/dsl/experiment/section.py`<br>`src/rust/laboneq-scheduler/src/analysis/validate_ir.rs` |
| Acquire loops         | Represent real-time averaging/acquisition boundaries, control repetition and oscillator phase resets     | `src/python/laboneq/dsl/experiment/acquire.py`<br>`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs` |
| Repetition modes      | Control timing between loop iterations, resolved during lowering                                         | `src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`                                     |
| Delay semantics       | Delays lowered to timed nodes, precompensation cleared with zero-duration nodes                           | `src/python/laboneq/compiler/workflow/precompensation_helpers.py`<br>`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs` |
| PRNG and match/case   | PRNG setup carried in sections, match/case lowered to timed conditional nodes                            | `src/rust/laboneq-scheduler/src/lower_experiment/match_case.rs`                             |
| Feedback constraints  | Enforced during validation, reject unsupported feedback or timing violations                             | `src/rust/laboneq-scheduler/src/analysis/validate_ir.rs`                                    |
| Validation passes     | Ensure timing, parameter, and hardware constraints are met before code generation                         | `src/rust/laboneq-scheduler/src/analysis/validate_ir.rs`                                    |

---

## 11. Mermaid diagram: Scheduling pipeline overview

```mermaid
flowchart TD
    A[Python DSL Experiment] --> B[ExperimentInfoBuilder]
    B --> C[Python Compiler Workflow]
    C --> D[Compatibility Bridge (build_rs_experiment)]
    D --> E[Rust DSL Experiment Tree]
    E --> F[Scheduler (schedule_experiment)]
    F --> G[ScheduledNode Tree (with constraints)]
    G --> H[Lowering Pass (to IrNode)]
    H --> I[Final Timed IR Tree]
    I --> J[Code Generator]
    J --> K[Device-specific Code & Artifacts]
```

---

## 12. Key source files and references

| Component                | Source file(s)                                                                                          | Description                                                                                      |
|-------------------------|-------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Python DSL Section       | [`src/python/laboneq/dsl/experiment/section.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py) | Defines `Section` class with alignment, timing mode, triggers, and play-after semantics         |
| ExperimentInfoBuilder    | [`src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py) | Builds compiler input structures from DSL and setup                                            |
| Scheduler (Python wrapper) | [`src/python/laboneq/compiler/scheduler/scheduler.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/scheduler/scheduler.py) | Python interface to Rust scheduler                                                             |
| Scheduler (Rust core)    | [`src/rust/laboneq-scheduler/src/scheduler.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs) | Main scheduling orchestration and passes                                                      |
| Lowering pass            | [`src/rust/laboneq-scheduler/src/lower_experiment/mod.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs) | Converts DSL operations to timed IR nodes                                                      |
| Validation               | [`src/rust/laboneq-scheduler/src/analysis/validate_ir.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/analysis/validate_ir.rs) | Validates timing, feedback, and parameter constraints                                         |
| Precompensation helpers  | [`src/python/laboneq/compiler/workflow/precompensation_helpers.py`](https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/precompensation_helpers.py) | Validates and clamps precompensation parameters                                               |
| PRNG and match/case      | [`src/rust/laboneq-scheduler/src/lower_experiment/match_case.rs`](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/match_case.rs) | Lowers conditional and PRNG constructs                                                       |

---

## 13. Invariants and constraints

- **Single AcquireLoopRt per experiment**: The experiment must contain at most one real-time acquisition loop.
- **Near-time sections not nested under real-time loops**: Near-time sections cannot be nested inside real-time loops.
- **Timing grid commensurability**: All signals and operations must be scheduled on compatible timing grids.
- **Strict timing sections disallow frequency sweeps and phase resets**: Hardware operations that consume time are forbidden in strict timing sections.
- **Feedback operations require validation**: Feedback-related nodes must satisfy device and timing constraints.
- **Precompensation clearing nodes have zero duration**: They do not affect timing but mark state resets.
- **Repetition modes must be consistent with loop structure**: The scheduler enforces repetition mode semantics to avoid timing conflicts.

---

## 14. Summary

The scheduling and timing subsystem in LabOne Q is a complex but well-structured pipeline that transforms high-level experiment descriptions into precise, timed instructions for hardware execution. It carefully manages timing grids, section semantics, acquisition loops, repetition modes, and hardware-specific constraints. The scheduler ensures that the compiled experiment respects all timing and device constraints, enabling reliable real-time quantum control.

Maintainers should understand the layered IR representations, the role of timing grids, and the semantics of sections and loops to effectively work on or extend this subsystem.

---

## References used on this page

1. LabOne Q repository, main page: https://github.com/zhinst/laboneq  
2. Python DSL Section: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/section.py  
3. ExperimentInfoBuilder: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/implementation/payload_builder/experiment_info_builder/experiment_info_builder.py  
4. Python Scheduler wrapper: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/scheduler/scheduler.py  
5. Rust Scheduler core: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/scheduler.rs  
6. Lowering pass: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/mod.rs  
7. Validation pass: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/analysis/validate_ir.rs  
8. Precompensation helpers: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/compiler/workflow/precompensation_helpers.py  
9. PRNG and match/case lowering: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-scheduler/src/lower_experiment/match_case.rs  
10. Python DSL AcquireLoopRt: https://github.com/zhinst/laboneq/blob/main/src/python/laboneq/dsl/experiment/acquire.py  
11. LabOne Q user manual, Core concepts: https://docs.zhinst.com/labone_q_user_manual/core/index.html  
12. LabOne Q Pulse Sheet Viewer: https://docs.zhinst.com/labone_q_user_manual/core/functionality_and_concepts/03_sections_pulses/concepts/02_pulse_sheet_viewer.html
