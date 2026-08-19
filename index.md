# Experimenting with Porting a Large C Library to Rust using Coding Agents

```{abstract}
With the rapid rise in abilities of Large Language Model Coding Agents such as Codex and Claude Code over the course of 2026, it became apparent that some software tasks which had seemed impossible to implement were now possible to contemplate.
In this document I will discuss one such task involving the conversion of a 250,000 lines of a C library that has been in development for 30 years to Rust.
The motivation for choosing Rust is the much stronger memory management model and additional type safety inherent in the language, coupled with the more modern tooling infrastructure available and the potential to support technologies such as WebAssembly.

The main outcome from this work is that your success in porting any code to a different language using agents depends entirely on the test coverage of your original source code.
```

## Introduction

In February 2026 I was modernizing the Python bindings of the Starlink AST library {cite:p}`2016A&C....15...33B` to make it usable as the WCS backend for our file format and data modeling rewrite of the LSST Science Pipeline images {cite:p}`DMTN-339`.
This was needed since there had been a new release of the C library that included critical functionality and we needed those changes [on PyPI](https://pypi.org/project/starlink-pyast/).
For historical reasons, the Python wrapping work started in 2011, the Python interface was written using native C Python APIs and had to support Python 2.6 and 3.x.
I decided to try using the newly-released OpenAI Codex with GPT-5.3 and I was impressed by its ability to quickly analyze the code, remove the Python 2 legacy interfaces and modernize the C API calls for Python 3.11 and newer.
Since I had long wondered whether the wrapper should be using Cython instead of native APIs, I asked Codex to make a proof of concept demonstration of a Cython wrapper and it did exactly that.
For context the hand-crafted C wrapper consisted of about 13,000 lines of code, so a full conversion was not a trivial undertaking but the demonstration did hint at future possibilities.
This proof of concept led to a discussion of the broader implications and we wondered if we could now port the AST C library to Rust.

The long-term support of the C library has been a concern in the project ever since we adopted the library for WCS in 2016 {cite:p}`DMTN-010`.
In particular the object-oriented C style and steep learning curve, associated with the library having a single developer, caused concern about the long-term support of the library[^longterm] and some people at Rubin had been wondering whether it would be better for the next decade if we had a Rust library implementing the core AST algorithms that we needed.

People have been wondering if large language models can aid with language ports for a while {cite:p}`10.48550/arXiv.2511.20617,10.1145/3729379` but the state of the art in this field is moving so fast that techniques and failures from 2025 using models like ChatGPT 4 are fully out of date.
In {cite:t}`10.48550/arXiv.2511.20617` they built a specialist infrastructure specifically for using LLMs to convert from C to Rust but the largest code base they tackled was of order 15,000 lines of code.
More recently, May 2026, there was a demonstration of a port of the 500k LOC `ngspice` package from C to Rust, [spice-rs](https://github.com/ferrite-systems/spice-rs).
The Git repository does not include any of the porting history and so it is difficult to say how the codebase evolved but the goal of the port was similar to the goals described here: a faithful port to Rust with byte-exact conformance with the original library.

A port of this size inevitably means that it's impossible to review every line of code and there is a danger that the entire port is effectively "vibe-coded" {cite:p}`10.1080/08874417.2026.2621186,10.48550/arXiv.2506.23253` as more and more code flies buy and you always accept each suggestion.
For a small throw-away application this is an acceptable approach, but it comes with risks when porting a large library that is intended for production use.

## The Starlink AST Library

The Starlink AST library is a mature C library that has been in development since 1997 {cite:p}`1998ASPC..145...41W`
The standard FITS approach for celestial coordinates only defines undistorted coordinates {cite:p}`2002A&A...395.1077C`.
Over the years there have been numerous approaches to support distortions in FITS headers with the ill-fated attempt at a standard {cite:p}`FITSWCSIV` and non-standard prooposals such as [TPV](https://fits.gsfc.nasa.gov/registry/tpvwcs/tpv.html) {cite:p}`TPVWCS`, TNX {cite:p}`TNXWCS` and ZPX {cite:p}`ZPXWCS`, with SIP {cite:p}`2005ASPC..347..491S` remaining as the only commonly implemented option, but none of the approaches allow a WCS to be refined and extended in a generalized manner.
The Starlink AST library was an attempt to bypass these competing hard-coded alternatives by defining a core set of transformations and allowing them to be combined in any order, serially or in parallel, without having to define a new standard set of FITS headers for every combination.

The library was written in C using object-oriented techniques involving virtual dispatch tables and object-like properties.
C++ was not used given the immaturity of the C++ infrastructure and standardization in the mid-1990s.
AST objects also support reference counting and lifetime tracking.
Additionally, to improve performance when rapidly creating and freeing many objects, an entire memory allocation layer was added.

The core of the C library (not including tests) consists of nearly 475,000 lines.
This is a substantial library to port to any language.
There are three vendored C libraries handling the spherical geometry and time transformations: wcsLib {cite:p}`2011ascl.soft08003C`, ERFA (the BSD-licensed version of SOFA), and PAL {cite:p}`2013ASPC..475..307J`.
For this work we decided to retain those vendored libraries (around 65k lines of C) in the Rust port, with the eventual plan to port the subset needed for AST to Rust when we were convinced that the port was stable and accurate.

```{figure} starlink-ast-loc.svg

How the Starlink C AST library has evolved over the last 30 years.
This graph shows the lines of C code (comments plus code) as a function of time and includes the main library and associated test code.
It does not include the Fortran test code.
The Git repository started in 1998 after 100,000 lines of code had already been written.
Gaps in the timeline note where at least 3 months had elapsed between commits.
```

## Previous Attempts at Porting the AST Library

In the early 2000s the Starlink Project was instructed to switch away from using C and Fortran with a bespoke message bus (ADAM) and instead switch to the more modern technologies of Java and SOAP {cite:p}`2003ASPC..295..445B`.
As part of that work it was realized that the AST library was a core foundation of the Starlink software ecosystem.
They started work on such a port in early 2002 but by October 2002 they had already decided that as an interim measure there should be a JNI wrapper around the existing AST C library.[^jniast]
This has many disadvantages in that it required a different binary per supported architecture but it did unblock the use of WCS in tools such as SPLAT {cite:p}`2014A&C.....7..108S`, SoG (Son of GAIA), and Treeview.[^treeview]
By the time of the 2003 ADASS {cite:p}`2004ASPC..314..412B` there was no mention of a native Java port, solely JNI, and the project had been abandoned.

There is no public record of the project's demise, but later discussions (Berry, private communication) indicated that the project failed due in part to the difficulty in porting all the legacy FITS header support in a reliable way and the huge scope of porting the complex C code manually.

## AST Usage in the LSST Science Pipelines

There were two aspects to integrating the AST library into the LSST Science Pipelines {cite:p}`PSTN-019`.
Firstly a C++ wrapper was written[^astshim] to provide an interface more similar to existing C++ semantics used in the pipelines code.
This included the addition of more controlled object lifetimes and standardized property accessors.

The second aspect was to wrap the AST internals in a more specialist `SkyWcs` C++ class such that a pipelines user would never know how the WCS transforms were implemented unless they were explicitly manipulating the mappings.
The AST plain text serialization was then used when writing the object to FITS.
This approach was also adopted by the `lsst-images` package {cite:p}`DMTN-339` where the AST internals (this time also backed by the PyPI Python wrapper) are abstracted behind a `SkyProjection` class and the plain text is stored in the JSON serialized model in files.

## Porting to Rust

As the C++ standard has become more and more complex, resulting in books actively telling you all the ways you can make mistakes {cite:p}`lakos2021embracing,yonts2025mistakes,meyers2014effective`, we increasingly sought alternatives.
At the Vera C. Rubin Observatory we have been considering the use of Rust {cite:p}`klabnik2026rust` for some time, with [RFC-1069](https://rubinobs.atlassian.net/browse/RFC-1069) in early 2025 approving its usage in the LSST Science Pipelines.
The first Rust code was added to the LSST Science Pipelines {cite:p}`PSTN-019` in early 2026[^rubinoxide]

There are numerous reasons to consider porting the C AST library to Rust.

1. The bespoke object-oriented implementation comes with a lot of boiler plate code and a steep learning curve.
   Rust includes native support for many equivalent concepts, raising the possibility of the same functionality using far less code and being more focused on the algorithms that matter.
2. Memory safety is a core goal of Rust with a very strong ownership and borrowing model making many coding errors relating to memory management and buffer overflows impossible.
3. Rust has a much stronger type system than C, enforced by the compiler, and has explicit approaches for indicating optional return values and error states that can be reasoned by the compiler and are much easier to track than the AST global status.

Additionally, once a decision has been made to port to Rust there are well-supported ways for providing a backwards compatible C interface as well as the possibility of having a clean independent Python wrapper directly on the Rust.
These reasons were enough to suggest that we should try to port the AST library, even though it is clearly a very challenging library to pick as a first test of using agents to do a port.

## The First Prototype

In order to investigate the feasibility of a port I decided the easiest thing to do was to try it and see what would happen.
My previous experience of using coding agents was for small fixes and code cleanups as described in the introduction.
Late in the afternoon of 2026-03-03 I decided to try the newly-released OpenAI Codex application (model 5.3) to start working on the port.
I was using a $20/month Team account for this work.
At the time I did not know enough to make sure there was an `AGENTS.md` file to refer to repository policies and was not aware that much can be gained by explicitly planning more up front.

My first prompt was:

> I am attempting to port the C library in the ast/ subdirectory to rust.
> A description of the plan with more instructions can be found in the file TODO.md.
>
> Please read that file and come up with an initial plan for the port.
> Append to TODO.md as needed to remember decisions.

and my associated support file was:

> The aim of this work is to convert the Starlink AST C library (in the ast directory) into a rust library in the starlink-astrs directory.
>
> The rust library should be object-oriented like the original and should follow all standard rust packaging and linting.
> It should also include C bindings so that it can replace the C version and to allow easy testing.
>
> The documentation describing the C library is found in sun211.pdf.
>
> The C code has a lot of standard boiler plate trying to emulate an object-oriented library in C without C++.
> Work was started on the package before C++ was mainstream.
> A full git history can be found in the ast/ subdirectory.
> The rust version can drop the boiler plate and focus on the business logic implementing the frames and mappings.
>
> You may append to the file in the PORT.md file with notes as needed to remember previous decisions

where I had made available the AST C source code in a parallel tree.
It then wrote an 80-line outline of a plan and started work.
By the evening of 2026-03-05 (two days later) I had 25,000 lines of Rust library code and 12,000 lines of tests and I had run out of my weekly allocation of tokens on the cheap Codex plan.
This seemed incredibly promising.
The coding agent was able to build the C library and write exploratory code against it to act as an "oracle" (its term) with which to compare the Rust code.

Work resumed towards the end of March and whilst initial progress was good, adding another 10,000 lines in a day, work then became much more difficult.
As the porting evolved I decided to focus on a core functionality of being able to read a Rubin-style FITS header with SIP distortions and run the transform chain for that end-to-end.
I had it write benchmarking tooling to compare performance of the polynomial distortions between C and Rust with different numbers of inputs to compare the performance of small numbers of points (C is relatively slow with those) versus large numbers of points (C uses extensive up front set up that is optimized for large numbers of points).

Towards the end of March there was a significant drop off in code generation as the agent tied itself in knots trying to work on simplification logic and this resulted in a decision to step back from the port and re-assess (once I had run out of tokens in two days).
It also turned out that March 2026 was special in that OpenAI had allocated significantly more tokens to users of the new Codex app than they would subsequently allow towards the end of April, leading to a false sense

```{figure} starlink-astrs-proto-loc.svg

Lines of Rust code as a function of time for the first Rust prototype.
The gap indicates time not working on the port.
```

### Lessons Learned

In about 10 days the port implemented about 15% of the C library's logic, dominated by code for reading a Rubin FITS header and transforming the resultant `FrameSet` from pixel coordinates to sky coordinates and vice versa.
Writing of FITS headers was not implemented but native plain text serialization as added for the supported subset of mappings.

The work did demonstrate one of the key motivations for the investigation.
We have long known that AST is extremely inefficient at transforming small numbers of points.
This is because there are set up costs in the C before a transformation can trigger but those are not necessary in Rust.
For the representative Rubin `FrameSet` a single point transform is 3- to 8x faster than the C.
For large numbers of points in some cases with the `PolyMap` transformation the Rust is 1.2x faster going forward and 1.5x going faster for the inverse, without using vectorization.
Clearly some of these optimizations for the large scale `PolyMap` improvements could also be applied to the C, but it did provide solid evidence that we would not lose performance in adopting Rust.
One key insight from this is that the ability to run tests and benchmarks directly against the C "oracle" is invaluable.
Codex built a full oracle test and benchmarking harness and was able to iterate rapidly on the profiling because of this.
Additionally it instrumented the C and Rust code to report exactly which trigonometry calculations were being triggered and could therefore prove that the C and Rust were doing the same number of calculations when they got the same answer, instead of the Rust getting the answer because of a heuristic.
One ongoing lesson from this work is that agents will cut corners to solve the immediate problem even if that causes a later problem.

When this work started it was an open-ended port where guidance was given to the agent interactively but there was only the one guiding principle of porting everything.
One decision I made early on was to record porting decisions in a single place (`PORT.md`), eventually having it be updated every time a decision was made or some problem was hit.
This provided a valuable historical record but no thought was given as to how it affected the agent's context -- the file was being scanned with every prompt and eventually that content dominated everything (this was before contexts in the millions were supported).
In retrospect it would have been better to have this document be used as an active record of the current state of the port rather than it continuing to grow as new information becomes available.
Having that file include the active to do list for the port plus any decisions made in the port and any mistakes made that should not be repeated, would have given a more bounded document that could iteratively improve the quality of the port.

There were a number of interesting problems encountered during the work:

1. When looking at a dual-sideband FITS header the Rust was not calculating the correct spectral values and tried multiple times to work out why there were differences, including optimizations involving irrelevant FITS header cards.
   Once the error was below 300 Hz it couldn't make any improvement and was at risk of saying that was close enough.
   It turned out that the real problem was that there was a missing timescale conversion from UTC to TDB earlier in the chain.
2. One of the key aspects of AST is its ability to simplify complex `FrameSets` into simpler forms by noting cases where two adjacent mappings can be combined (such as a forward mapping and an inverse mapping canceling out, or a `ZoomMap` and a `MatrixMap` being combined into a single `MatrixMap`).
   This is critical for performance since it minimizes the number of transforms to be calculated.
   When I asked Codex to add simplification I failed to notice that, even though it mentioned in the porting document that the C library has simplification logic per mapping, it was putting special-cased simplification logic in a single file.
   The only clue was firstly that it was having a harder and harder time matching the simplification logic from the C as it stacked more and more variations in one place, but also that the `cmp_map.rs` file was becoming significantly larger than the C original.
   This was an example where the agents are by default instructed to do the minimum work to satisfy the goal, and adding some specialist simplifications in the main file was less effort than setting up an entirely new multi-mapping infrastructure.
3. At no point did I explain the `AST__BAD` magic value to the agent, and so it mostly failed to add support for it in the transform code path.
   This was possible because at the time there was no C test code that triggered the problem and so there was no motivation to implement it.
   Codex did eventually come up with a checksum approach for the oracle transform testing to allow it to do many transformations and quickly spot problems such as the missing `AST__BAD` support.

In addition to these issues there were some more serious examples of the agent faking test results.

1. For the FITS header tests the agent knew that the idea was to read in the FITS header, convert it to a `FrameSet` and then write it out in the native plain text serialization and compare it with the oracle.
   The test succeeded because there was specialist code added that on read would recognize which FITS header it was, inject the serialized oracle form into an internal cache, and then on write it would write out the cached text.
   This is clearly the easiest way to pass the test but over the 10 days Codex was caught multiple times trying to cache a pre-canned answer so it could pass the test.
2. In another example Codex rewrote the serialized form to make it look like the correct oracle form.
3. At one point some test fixtures that were meant to be generated by the oracle were in fact generated by the Rust -- a guarantee that the test would pass.
4. On another occasion the Rust code started to special case different combinations of spatial and spectral FITS headers since it did not understand that in most cases parsing the spatial WCS is distinct from parsing the spectral WCS.
   It also created an entirely new class, `TanSkyMap` specifically to handle one specialist FITS header.
5. The `degen1.ast` example `FrameSet` is one of the trickiest to handle since it has a degenerative WCS with mismatched pixel axes, the rust test code was failing to generate a simplified form that matched the C and so it had a special clause in the code specifically for this `FrameSet` to reorganize it in the required form.
   The workaround was explicitly turned off when reading similar example files that would be broken by the rewrite.
6. Special-cased handling was also included for some form of CAR headers.

There were also inefficiencies:

1. When parsing a SIP FITS header, the code was creating a serialized native text form of the polynomial definition and handing it to `PolyMap` to be parsed rather than just using the direct `PolyMap` constructor.
2. I wondered about performance issues too early in the process and driving the agent to make something faster led to it putting optimizations in the wrong places which then led to downstream confusion that built on those mistakes.

The fundamental learning from the prototype work was that test coverage of the C library is critical, as is the use of the oracle to provide immediate feedback to compare with the new code when there is a disagreement.
At the time test coverage in AST was mainly handled through Fortran testing with no formal coverage reporting.
This then provided the focus for the next phase.

## Tests Are Important

By April 2026 it was realized that there were some fundamental work that could be done on the C library that could improve the quality of the Rust port.

The first task was to simplify the build system.
The AST C library uses the GNU Autoconf tooling but with Starlink extensions for handling some of the Fortran interfaces {cite:p}`2005ASPC..347..119G`.
The core library is not in itself complicated to build, but the Git source has no means to build it automatically without having access to the Starlink software distribution.
For the prototype we got around this by teaching Codex how to build the oracle based on it working out how the Python bindings were built.
This was not the most efficient approach and so in April 2026 we used Claude Opus 4.6 to build an entire parallel CMake build system that would build the C library (not the Fortran bindings) and run the C tests.
We also methodically converted the Fortran tests to C and added coverage reporting into CI so we could determine the current state of the test suite.
For one of these Fortran test files (about 1300 lines of Fortran) I had run out of Claude tokens and switched to Gemini Pro 3.1 -- it took a couple of hours and didn't make much progress.
When I had access to Claude it looked at the code, determined that the code was a very naive direct Fortran-to-C port, and decided it was easier to rewrite it from scratch, which it did in ten minutes.
This work was focused on determining the current code coverage as a whole and not specifically the parts that we wanted to focus on for the initial port, and so included coverage for the plotting subsystem.
Since the Fortran plotting tests required PGPLOT we added a new graphics backend for PLplot and additionally Claude wrote a native SVG plotting backend which was very useful since Claude can read SVG outputs natively and reason about their contents, allowing it to iterate rapidly to ensure that the SVG was rendering correctly.
When this code was merge the code coverage for the core library code was approximately 50%.
Adding GitHub Actions CI also enabled us to easily do sanitizer builds, which immediately found a couple of latent buffer overruns that had been around for a long time.

Once that was in place the next step was to create more example FITS headers.
For this work Claude was tasked with improving the coverage solely of the FITS handling file.
It was able to read the code and then attempt to create a FITS header targeting a specific line.
The FITS header handling is one of the most esoteric part of the AST source since it encodes many historical variants of FITS headers, many of which are not in wide circulation today, but it was felt that a port that could only read current FITS WCS variants would not be a real port, and we could not abandon the historical support (the AST library is used by the SAOImageDS9 image viewer specifically for this reason).
This work added 90 new FITS headers to the test fixtures, with the code coverage for the FITS handler increasing to 75% -- the remaining coverage was difficult to establish since much of it involves correctness tests that are acting as safety nets confirming a situation that is already handled earlier in the chain.

This work provided the foundation for the next porting attempt.

## Starting Again From Scratch

Once we had a better build and test story for the C library, by late April 2026 we decided to make another attempt at doing the port from scratch.
By this time Claude Opus 4.6 was the current model and in addition to a $20/month team account I had access to a Claude API key through the DOE SLAC National Laboratory near Stanford.
Since Rubin is partially funded by DOE we were allowed to make use of their new infrastructure based on Amazon Bedrock since SLAC were instructed to investigate ways in which LLMs can be used in research activities.
For FY26 SLAC are absorbing the cost of the API keys as part of the research experiment so Rubin was able to make use of this without being charged.

This time we decided to start by planning what we thought the port should look like in terms of memory management and mutability and how traits should relate to C objects.
As was done previously the wcsLib, ERFA, and PAL source code were copied unchanged with Rust shims on top.

### Lessons Learned

For the AST port adding more simplification fixtures led to a critical discovery that the `CmpMap` flattening in Rust was using a different (but mathematically equivalent) algorithm to the C code -- transforms were working fine before and after simplification but the mappings had a different shape when serialized.
Fixing that took a while to recover with many commits porting C algorithms.

## Recommendations

```{figure} starlink-port-interleave.svg

A timeline for the work described in this document as of mid August 2026.
You can see how the prototype triggered work in the C library to update the build system and add test coverage and that later work on the port resulted in separate work on the C library (bug fixes and additional test fixtures).
The spike in the AST commit history in May is adding many more simplification example test fixtures.
```

The key takeaway from this work is that your language port is only as good as the test coverage of your original code.
The coding agent will try to take the easiest path to generating the code from your prompt.
You can ask it to be faithful to the source language but you can't be sure it is really going to do that unless there are tests that demonstrate exactly what you need.
For AST this is made more explicit by the addition of hundreds of test fixtures involving serialized FrameSets and Mappings as both native AST serializations and FITS headers.
Adding test coverage to the source library before you start the port, especially for edge cases, pays dividends many times over once you start on the port.
You can use a coding agent to add the tests.

* Always write a plan first and refer to that plan in your `CLAUDE.md` or equivalent so that the plan is checked against implementation.
  Allow the plan to be updated based on lessons learned.
* Keep a running ledger of open work so that you do not continually have to scan the C and Rust source to check what has been ported.
  * You will want to port every test so for auditing purposes annotate in your new tests where the test came from originally.
  * Consider having a TODO list of every test that you can then mark off as the port progresses.
* Correctness and completeness are more important for a port than performance.
  Do not start optimizing until you are passing every test.
  Even if you have code coverage for the small area you are optimizing you make it harder later on to reason about whether the Rust version truly matches the C approach and some times the optimization can shift logic around that confuses a later code addition that relied on the earlier form.
* Ensure that your coding standards can not be bypassed.
  Use `pre-commit` / `prek` hooks to capture all code linting such as `ruff` and `numpydoc` in Python and `clippy` in Rust.
  Every gate you can apply to commits forces the agent to follow your guidelines in a far more robust way than making a note in the `CLAUDE.md` file ever would.
  In Python use as many `ruff check` options as you can.
* Some coding practices do have to be included in the `CLAUDE.md` file but remember that this is guidance and not treated as a hard rule --- the prompt and history will all be processed along with the `CLAUDE.md`.
  Negative directives are weaker than positive directives.
  For Python, agents have a tendency to prefer to add imports locally rather than at the top of the file so consider adding an instruction to always put imports at the top unless there is an import cycle.
* For the AST port it was critical to strongly declare that the C algorithms were the source of truth and were to always be used as is and to only use heuristics in the Rust if the same heuristic was present in the C.
  The C library is the "oracle" and you have to insist that whenever there is a difficulty in getting a Rust test to pass that the oracle be checked and instrumented.
* In some cases there will be bugs in your original library and these will be noticed by the coding agent.
  Sometimes the agent will not tell you about them (such as triggering a SEGV in an oracle investigation) so you have to ask.
  Sometimes the agent will realize there is a bug in the original but code that bug into the port.
  I eventually added a standard instruction noting that bugs are possible in the original and should be recorded in a separate document.

[^astshim]: <https://github.com/lsst/astshim>
[^rubinoxide]: <https://github.com/lsst/rubinoxide>
[^jniast]: The JNIAST package in 2002 October included the lines "This package is a temporary measure; in due course it will be superseded by the pure java WCS package.  WCS is being actively worked on now (mid 2002), but the timescale for completion is uncertain."
[^treeview]: The [treeview docs](https://github.com/Starlink/starjava/blob/master/treeview/src/resources/uk/ac/starlink/treeview/docs/homepage.html#L313C5-L313C37) still refer to a todo item of "Pure java version of WCS support" as of August 2026.
[^longterm]: Ironically, the same argument for porting to Rust using coding agents could apply to using those same agents to update the existing C library --- by default the agents will match the coding style of the surrounding code and they are not scared by non-standard inheritance models or clever memory management.

## References

```{bibliography}
  :style: lsst_aa
```

## Appendix: Initial Port Design Spec

This is the initial design spec for the overall shape of the port from C as developed iteratively with the Superpowers plugin brainstorming mode on 2026-04-20 with input from David Berry and Jim Bosch.
Some things have changed

```{include} 2026-04-20-rust-port-design.md
```
