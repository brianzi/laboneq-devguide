# LabOne Q Developer Guide

This repository contains a MkDocs documentation site explaining the internal architecture of [`zhinst/laboneq`](https://github.com/zhinst/laboneq) for developers. It was prepared from an inspected public checkout of LabOne Q, related Zurich Instruments repositories, official Zurich Instruments documentation, and PyPI package metadata.

This is an **unofficial developer-oriented guide**. It is intended to complement, not replace, the official LabOne Q user manual and Zurich Instruments device manuals.

## Contents

The guide covers the LabOne Q ecosystem, repository layout, Python DSL frontend, compiler payload construction, Rust-backed compiler core, intermediate representations, scheduling and timing semantics, code-generation artifacts, runtime/controller execution, device communication layers, result data structures, and likely extension points for maintainers.

The Markdown source is in [`docs/`](docs/), and the site configuration is in [`mkdocs.yml`](mkdocs.yml). The guide includes source links to the upstream LabOne Q repository throughout the text.

## Build locally

```bash
python -m pip install mkdocs mkdocs-material pymdown-extensions
mkdocs serve
```

For a strict production build, run:

```bash
mkdocs build --strict
```

## GitHub Pages

A GitHub Actions workflow is included at [`.github/workflows/pages.yml`](.github/workflows/pages.yml). If GitHub Pages is enabled for this repository with **GitHub Actions** as the source, pushes to `main` will build and deploy the MkDocs site automatically.

## Scope note

The guide describes the public repository state inspected during preparation. LabOne Q evolves quickly, and Zurich Instruments may change internal APIs, Rust/Python boundaries, generated artifacts, or compatibility layers without preserving the exact internal shapes documented here. Treat the guide as a map for code reading and maintenance rather than a compatibility contract.
