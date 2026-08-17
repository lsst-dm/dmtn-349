[![Website](https://img.shields.io/badge/dmtn--349-lsst.io-brightgreen.svg)](https://dmtn-349.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-349/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-349/actions/workflows/ci.yaml)

# Experimenting with Porting a Large C Library to Rust using Coding Agents

## DMTN-349

With the rapid rise in abilities of Large Language Model Coding Agents such as Codex and Claude Code over the course of 2026, it became apparent that some software tasks which had seemed impossible to implement were now possible to contemplate.
In this document I will discuss one such task involving the conversion of a 250,000 lines of a C library that has been in development for 30 years to Rust.
The motivation for choosing Rust is the much stronger memory management model and additional type safety inherent in the language, coupled with the more modern tooling infrastructure available and the potential to support technologies such as WebAssembly.

The main outcome from this work is that your success in porting any code to a different language using agents depends entirely on the test coverage of your original source code.

**Links:**

- Publication URL: https://dmtn-349.lsst.io
- Alternative editions: https://dmtn-349.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-349
- Build system: https://github.com/lsst-dm/dmtn-349/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-349
cd dmtn-349
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-349.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-349.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
