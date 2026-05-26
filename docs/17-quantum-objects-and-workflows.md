# Higher-level application layers: quantum objects, operations, and workflows

The preceding chapters follow LabOne Q downward from the Python DSL into scheduling, lowering, code generation, runtime execution, and result objects. This page re-enters the stack from above. **Quantum objects**, **quantum operations**, and **workflows** are the application-facing layers that help users describe calibrated devices, reuse experiment-building idioms, and orchestrate multi-step campaigns. They do not replace the compiler. They produce and consume the same DSL experiments, compiled artifacts, runtime calls, and results that the lower chapters describe.

```mermaid
graph TD
    A[Calibrated device knowledge] --> B[Quantum elements and QPU]
    B --> C[Quantum operations]
    C --> D[Ordinary DSL experiment]
    D --> E[Payload builder]
    E --> F[Compiler pipeline and IRs]
    F --> G[Compiled artifacts]
    G --> H[Runtime execution]
    H --> I[Results]
    I --> J[Workflow analysis and update tasks]
    J --> B

    B -. detailed page .-> B1[Quantum elements and QPU]
    C -. detailed page .-> C1[Quantum operations]
    J -. detailed page .-> J1[Workflows]
```

This layer is easiest to understand as three cooperating abstractions. A `QuantumElement` stores the calibrated state and signal vocabulary of a device object such as a qubit or coupler. A `QPU` collects those elements, binds a compatible `QuantumOperations` instance, and carries topology. A workflow records and executes a task graph that can build, compile, run, analyze, and update experiments.

| Layer | Main abstraction | Produces or coordinates | Detailed page |
| --- | --- | --- | --- |
| Device knowledge | `QuantumElement`, `QuantumParameters`, `QPU`, `QPUTopology` | Signal names, parameter state, topology, and QPU-bound operation sets. | [Quantum elements and QPU](17a-quantum-elements-and-qpu.md) |
| Experiment macros | `QuantumOperations`, `Operation`, `@quantum_operation` | DSL sections, reserves, pulses, acquisitions, and nested operation calls. | [Quantum operations](17b-quantum-operations.md) |
| Campaign orchestration | `@workflow`, `@task`, workflow blocks, executor state, workflow results | Ordered compile/run/analysis/update procedures over ordinary LabOne Q APIs. | [Workflows](17c-workflows.md) |

## The central boundary

The application layer is **above the compiler frontend**. It helps construct a DSL `Experiment`, a `DeviceSetup`, calibration data, and task calls, but the compiler still receives explicit frontend objects. A quantum operation call therefore belongs to the same semantic side of the stack as a handwritten `with experiment.section(...):` block: Python executes the helper, LabOne Q records ordinary DSL content, and the compiler later schedules that content.

```mermaid
sequenceDiagram
    participant User as User code or workflow task
    participant QPU as QPU and quantum elements
    participant QOps as QuantumOperations
    participant DSL as DSL experiment builder
    participant Compiler as Compiler pipeline
    participant Runtime as Runtime controller

    User->>QPU: select elements and parameter state
    User->>QOps: call operation, for example x90(q0)
    QOps->>DSL: create section, reserve signals, play pulse
    User->>Compiler: compile experiment
    Compiler->>Compiler: schedule, lower, generate artifacts
    User->>Runtime: run compiled experiment
    Runtime->>User: results for workflow analysis
```

The important consequence is that debugging remains boundary-oriented. If a QPU experiment builds the wrong pulse, inspect the quantum operation implementation and element parameters. If the emitted DSL looks correct but compilation fails, inspect the compiler-input payload and IRs. If compilation succeeds but execution or data collection fails, inspect runtime handoff and result processing. The same artifacts used in the lower guide remain the useful artifacts here.

| Symptom | Most likely boundary | First artifact to inspect |
| --- | --- | --- |
| Operation emits the wrong signal or pulse parameter | Quantum element or quantum operation layer | Generated DSL experiment, operation source, and element parameter snapshot |
| High-level experiment compiles differently than expected | Frontend-to-compiler boundary | Payload builder input and scheduled experiment IR |
| Compiled waves or recipe look wrong | Backend and artifact layer | Generated waves, recipe, command tables, and artifact metadata |
| Workflow step updates calibration incorrectly | Workflow and analysis layer | Workflow result tree, task inputs, analysis outputs, and QPU update arguments |
| Re-running a notebook campaign differs from saved execution | Persistence and orchestration layer | Workflow result, serialized experiment or compiled experiment, and saved quantum-object state |

## Three timescales

The abstractions above the compiler also separate the timescale of a calibration campaign from the timescale of a single compile or run. Quantum elements persist calibrated knowledge across many experiments. Quantum operations instantiate that knowledge into a specific experiment. Workflows coordinate repeated execution and analysis, often feeding results back into the next QPU state.

```mermaid
graph LR
    A[Long-lived calibration model] --> B[Experiment construction]
    B --> C[Single compile]
    C --> D[Single execution]
    D --> E[Analysis result]
    E --> A

    A:::slow
    B:::build
    C:::compile
    D:::run
    E:::slow

    classDef slow fill:#eef7ff,stroke:#5b8def,color:#1f3763
    classDef build fill:#f7f3ff,stroke:#8264d6,color:#3b2868
    classDef compile fill:#fff7e6,stroke:#d89a32,color:#6d4713
    classDef run fill:#eefaf1,stroke:#58a55c,color:#204d24
```

This framing keeps the section intentionally high-level. The detailed pages that follow explain the main application-layer objects without duplicating the compiler pipeline chapters.

## Maintainer reading order

Start with [Quantum elements and QPU](17a-quantum-elements-and-qpu.md) when the change concerns parameter containers, signal names, topology, or QPU copying and updates. Read [Quantum operations](17b-quantum-operations.md) when the change concerns operation registration, dispatch, section wrapping, broadcasting, signal reservation, or operation composition. Read [Workflows](17c-workflows.md) when the change concerns task graph construction, task execution, options propagation, workflow results, or standard compile/run tasks.

The source-level map for these areas is kept in the appendix-oriented [source reference](15-source-reference.md), not in this overview. That separation is deliberate: the early pages should explain the mental model, while the appendix should serve as the file-level lookup table.
