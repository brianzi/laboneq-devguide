# Dependencies and build system

This page provides a comprehensive overview of the dependencies and build system of the LabOne Q project (`zhinst/laboneq`). It covers the Python packaging infrastructure, the Rust extension built with maturin and PyO3, the core third-party dependencies including Zurich Instruments communication libraries, and the optional dependency groups. The discussion includes practical guidance for developers on what dependencies exist, why they are needed, where they are located in the repository, who consumes them, and the implications for building and testing the project.

---

## How to read this page as a maintainer

This page is structured to guide maintainers through the dependency landscape and build system of LabOne Q. It begins with the Python packaging configuration (`pyproject.toml`), explaining how the Rust extension is integrated via maturin and PyO3. It then details the core third-party dependencies, especially those related to Zurich Instruments hardware communication, and explains the rationale behind optional dependency groups. Finally, it discusses how these dependencies affect the build and test processes.

Maintainers should use this page as a reference when updating dependencies, modifying the build system, or troubleshooting build and runtime issues related to dependencies. The page includes links to relevant source files and configuration files to facilitate deeper exploration.

---

## 1. Python packaging: `pyproject.toml`

The LabOne Q repository uses [PEP 517/518](https://peps.python.org/pep-0517/) compliant packaging with a `pyproject.toml` file located at the repository root ([source](https://github.com/zhinst/laboneq/blob/main/pyproject.toml)). This file governs the build system requirements, project metadata, dependencies, and optional dependency groups.

### 1.1 Build system configuration

The `pyproject.toml` specifies `maturin` as the build backend. Maturin is a tool designed to build and publish Rust-based Python extensions using PyO3 or rust-cpython bindings.

```toml
[build-system]
requires = ["maturin>=0.13,<0.15"]
build-backend = "maturin"
```

This configuration instructs Python packaging tools (e.g., `pip`) to use maturin to build the Rust extension during installation or wheel building.

### 1.2 Project metadata and dependencies

The `pyproject.toml` declares the project name, version, authorship, and core dependencies. The core dependencies include:

- `zhinst-comms`
- `zhinst-core`
- `zhinst-toolkit`
- `attrs`
- `numpy`
- `typing-extensions`
- `capnp` (Cap'n Proto Python bindings)

These dependencies are essential for LabOne Q's runtime and compilation workflows, particularly for communicating with Zurich Instruments hardware and handling experiment data structures.

### 1.3 Optional dependency groups

The project defines optional dependency groups to allow users and developers to install subsets of dependencies tailored to specific use cases:

| Group Name | Purpose |
|------------|---------|
| `dev`      | Development dependencies including linters, formatters, and testing tools. |
| `docs`     | Dependencies required for building documentation. |
| `examples` | Dependencies needed for running example notebooks and scripts. |
| `test`     | Testing frameworks and coverage tools. |
| `extras`   | Additional features or integrations that are not core but useful for extended functionality. |

These groups are declared under the `[project.optional-dependencies]` section in `pyproject.toml`. For example, the `dev` group includes `pytest`, `black`, and `mypy`.

---

## 2. Rust extension: maturin and PyO3 integration

LabOne Q integrates a Rust compiler and scheduler backend as a Python extension module. This Rust extension is built using [maturin](https://github.com/PyO3/maturin) and [PyO3](https://pyo3.rs/), allowing seamless interoperability between Python and Rust.

### 2.1 Location and structure

The Rust extension source code resides primarily under:

- `src/rust/laboneq-rust/` — the main Rust crate exposing the PyO3 extension ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs))
- `src/rust/laboneq-compiler-py/` — the Rust compiler Python bridge crate ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs))
- Other Rust crates supporting IR, scheduler, DSL, and backend logic (e.g., `laboneq-ir`, `laboneq-scheduler`, `laboneq-qccs-backend`)

The Rust extension is exposed as a single Python module `laboneq._rust` with submodules `codegenerator` and `compiler` registered dynamically at runtime to avoid multiple extension binaries.

### 2.2 Build process

Maturin handles building the Rust extension as a Python wheel or source distribution. It compiles the Rust crates with the appropriate features and links them into a Python extension module.

The build process is triggered automatically during Python package installation or can be invoked manually via:

```bash
maturin develop
```

or

```bash
maturin build
```

for local development and packaging, respectively.

### 2.3 PyO3 bindings and API exposure

The Rust extension exposes several key functions and types to Python, including:

- `build_experiment_capnp()` — converts Python experiment info into Rust IR and compiles it ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs))
- `schedule_experiment()` — runs the scheduler on the Rust IR
- `generate_code()` — generates device-specific code artifacts from the scheduled IR ([source](https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs))

These bindings enable the Python compiler workflow to delegate heavy-lifting compilation and scheduling tasks to Rust, improving performance and safety.

---

## 3. Core third-party dependencies

LabOne Q depends on several third-party Python packages, primarily for hardware communication, data handling, and serialization.

### 3.1 Zurich Instruments communication stack

LabOne Q relies on the following Zurich Instruments Python packages:

| Package       | Purpose                                                                                  | Notes                                                                                  |
|---------------|------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| `zhinst-comms`| Protocol stack for communicating with the LabOne data server                            | Not intended for direct use; used internally by `zhinst-core` and LabOne Q             |
| `zhinst-core` | Native Python API for LabOne, enabling device communication                             | Binary extension; requires LabOne software installed locally                           |
| `zhinst-toolkit` | High-level driver package built on `zhinst-core` providing a pythonic interface      | Runtime dependency for LabOne Q controller/session; recommended for device access      |

These packages are essential for runtime communication with Zurich Instruments hardware and the LabOne data server. They abstract low-level protocol details and provide a stable API for device control.

### 3.2 Other core dependencies

- `attrs`: Used extensively in the Python DSL and data classes for concise and efficient attribute management.
- `numpy`: Fundamental for numerical operations, waveform data handling, and array manipulations.
- `capnp`: Python bindings for Cap'n Proto serialization, used for efficient data exchange between Python and Rust components.

---

## 4. Optional dependency groups and their implications

LabOne Q defines optional dependency groups to modularize dependencies and reduce installation overhead for users who do not require all features.

### 4.1 Dependency groups overview

| Group      | Typical packages included                         | Usage context                          |
|------------|-------------------------------------------------|--------------------------------------|
| `dev`      | `pytest`, `black`, `mypy`, `flake8`              | Development, testing, linting         |
| `docs`     | `mkdocs`, `mkdocs-material`, `pymdown-extensions` | Documentation building                |
| `examples` | `jupyter`, `matplotlib`, `notebook`              | Running example notebooks and demos  |
| `test`     | `pytest`, `pytest-cov`                            | Running test suites                   |
| `extras`   | Additional optional features or integrations     | Extended functionality, e.g., alternative backends |

### 4.2 Build and test implications

- Installing with `pip install laboneq[dev]` ensures all development tools are available for running tests and formatting code.
- The `docs` group is required for building the documentation site with MkDocs Material.
- The `examples` group allows running example notebooks that may depend on visualization or notebook packages.
- The `test` group is used in continuous integration pipelines to run unit and integration tests with coverage reporting.

Maintainers should ensure that changes to dependencies are reflected in the appropriate groups and that CI workflows install the necessary groups for the tasks they perform.

---

## 5. Repository layout related to dependencies and build

The repository layout reflects the separation of concerns between Python code, Rust crates, and build metadata.

| Directory / File                      | Purpose                                                             | Notes                                                                                   |
|-------------------------------------|---------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| `pyproject.toml`                    | Python packaging and build system configuration                     | Declares build backend, dependencies, and optional groups                               |
| `src/python/laboneq`                | Main Python package source code                                     | Contains DSL, compiler orchestration, controller, runtime, and utilities                |
| `src/python/laboneq/_rust`          | Python bindings for Rust extension                                  | Submodules `codegenerator` and `compiler` expose Rust functionality                     |
| `src/rust/laboneq-rust`             | Rust crate implementing the PyO3 Python extension                   | Entry point for Rust compiler and code generator                                       |
| `src/rust/laboneq-compiler-py`      | Rust crate bridging Python and Rust compiler internals              | Handles Cap'n Proto serialization and backend-specific compilation                      |
| `src/rust/laboneq-ir`               | Rust crate defining intermediate representations (IR)               | Central IR data structures for scheduling and code generation                           |
| `src/rust/laboneq-scheduler`        | Rust crate implementing the scheduling passes and lowering           | Converts DSL trees into timed IR nodes                                                 |
| `src/rust/laboneq-qccs-backend`     | Rust crate implementing hardware-specific backend preprocessing      | Maps logical signals to physical devices and applies device-specific rules             |

---

## 6. Dependency invariants and consumption

### 6.1 Python dependencies

- The Python DSL, compiler orchestration, and controller consume core dependencies such as `attrs`, `numpy`, and Zurich Instruments communication packages.
- The Python compiler workflow depends on the Rust extension via `laboneq._rust.compiler` and `laboneq._rust.codegenerator`.
- Cap'n Proto bindings (`capnp`) are used to serialize experiment data structures between Python and Rust.

### 6.2 Rust dependencies

- The Rust crates depend on internal crates such as `laboneq-ir` for IR definitions and `laboneq-scheduler` for scheduling logic.
- The Rust extension uses PyO3 to expose APIs to Python.
- The `laboneq-qccs-backend` crate depends on device-specific knowledge and enforces hardware constraints.

### 6.3 Zurich Instruments dependencies

- `zhinst-comms` and `zhinst-core` are runtime dependencies required for device communication.
- `zhinst-toolkit` is a higher-level runtime dependency used by the controller and session layers to manage instruments.

---

## 7. Build and test workflow implications

### 7.1 Building the project

- The Rust extension is built automatically during Python package installation via maturin.
- Developers can build the Rust extension separately using `maturin develop` for faster iteration.
- The Python package installation resolves and installs core and optional dependencies as specified in `pyproject.toml`.

### 7.2 Testing

- Tests are located primarily under `src/python/laboneq/testing` and related subdirectories.
- The `dev` and `test` optional dependency groups include testing frameworks such as `pytest`.
- Continuous integration pipelines install these groups to run tests and report coverage.

### 7.3 Documentation

- Documentation dependencies are installed via the `docs` optional group.
- The documentation build uses MkDocs Material and related plugins.

---

## 8. Summary table of key dependencies

| Dependency         | Type           | Purpose                                         | Location / Usage                             |
|--------------------|----------------|------------------------------------------------|----------------------------------------------|
| `maturin`          | Build tool     | Build Rust extension as Python module           | Build backend specified in `pyproject.toml` |
| `PyO3`             | Rust binding   | Bind Rust code to Python                         | Used in `laboneq-rust` and `laboneq-compiler-py` crates |
| `zhinst-comms`     | Python package | Protocol stack for LabOne data server communication | Runtime dependency for device communication  |
| `zhinst-core`      | Python package | Native LabOne Python API                         | Runtime dependency for device communication  |
| `zhinst-toolkit`   | Python package | High-level driver for Zurich Instruments devices | Runtime dependency for controller/session    |
| `attrs`            | Python package | Data class and attribute management              | Used extensively in Python DSL and data classes |
| `numpy`            | Python package | Numerical computing and array handling           | Used for waveform data and numerical operations |
| `capnp`            | Python package | Cap'n Proto serialization bindings               | Used for Python-Rust data exchange            |
| `pytest`           | Python package | Testing framework                                 | Included in `dev` and `test` groups           |
| `mkdocs-material`  | Python package | Documentation building                            | Included in `docs` group                       |

---

## 9. Mermaid diagram: Dependency and build system overview

```mermaid
graph TD
    A[pyproject.toml] -->|Build backend| B[maturin]
    B --> C[Rust extension (laboneq._rust)]
    C --> D[laboneq-rust crate]
    C --> E[laboneq-compiler-py crate]
    D --> F[laboneq-ir crate]
    D --> G[laboneq-scheduler crate]
    D --> H[laboneq-qccs-backend crate]

    A --> I[Python dependencies]
    I --> J[zhinst-comms]
    I --> K[zhinst-core]
    I --> L[zhinst-toolkit]
    I --> M[attrs]
    I --> N[numpy]
    I --> O[capnp]

    I --> P[Optional groups]
    P --> Q[dev (pytest, black, mypy)]
    P --> R[docs (mkdocs-material)]
    P --> S[examples (jupyter, matplotlib)]
    P --> T[test (pytest-cov)]

    subgraph Python Package
        U[laboneq Python package]
        U --> I
        U --> C
    end
```

---

## 10. Practical developer orientation

### 10.1 What exists

- A Python package with a Rust extension built via maturin and PyO3.
- Core dependencies for hardware communication (`zhinst-comms`, `zhinst-core`, `zhinst-toolkit`).
- Optional dependency groups for development, testing, documentation, and examples.
- A layered Rust crate structure supporting IR, scheduling, backend preprocessing, and code generation.

### 10.2 Why it exists

- The Rust extension provides performance-critical compilation and scheduling functionality.
- Zurich Instruments dependencies enable runtime communication with hardware.
- Optional groups modularize dependencies for different developer and user needs.
- The build system ensures seamless integration of Rust and Python components.

### 10.3 Where it lives

- `pyproject.toml` at the repository root governs packaging and dependencies.
- Rust crates under `src/rust/` implement the compiler backend.
- Python source under `src/python/laboneq/` includes DSL, compiler orchestration, controller, and runtime.
- Rust extension Python bindings under `src/python/laboneq/_rust/`.

### 10.4 Who consumes it

- Python users and developers installing LabOne Q consume the Python package and Rust extension.
- The compiler workflow consumes the Rust extension APIs.
- The controller and session layers consume Zurich Instruments communication packages.
- Developers use optional groups for testing, documentation, and examples.

### 10.5 What invariants it carries

- The Rust extension must be built with maturin and compatible with the Python package version.
- Zurich Instruments dependencies require compatible LabOne software installed locally.
- Optional groups must be kept up to date with development and CI requirements.
- The build system must ensure reproducible builds and consistent dependency resolution.

---

## References used on this page

1. LabOne Q repository, `pyproject.toml`: https://github.com/zhinst/laboneq/blob/main/pyproject.toml  
2. LabOne Q repository, Rust extension root: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-rust/src/lib.rs  
3. LabOne Q repository, Rust compiler Python bridge: https://github.com/zhinst/laboneq/blob/main/src/rust/laboneq-compiler-py/src/lib.rs  
4. Zurich Instruments PyPI packages:  
   - https://pypi.org/project/zhinst-comms/  
   - https://pypi.org/project/zhinst-core/  
   - https://pypi.org/project/zhinst-toolkit/  
5. LabOne Q architecture notes from README: https://github.com/zhinst/laboneq/blob/main/README.md  
6. LabOne Q user manual: https://docs.zhinst.com/labone_q_user_manual/
