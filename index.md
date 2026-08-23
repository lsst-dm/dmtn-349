# Experimenting with Porting a Large C Library to Rust using Coding Agents

```{abstract}
With the rapid rise in abilities of Large Language Model Coding Agents such as Codex and Claude Code over the course of 2026, it became apparent that some software tasks which had seemed impossible to implement were now possible to contemplate.
In this document I will discuss one such task involving the conversion of 475,000 lines of a C library that has been in development for 30 years to Rust.
The motivation for choosing Rust is the much stronger memory management model and additional type safety inherent in the language, coupled with the more modern tooling infrastructure available and the potential to support technologies such as WebAssembly.

The main outcome from this work is that your success in porting any code to a different language using agents depends entirely on the test coverage of your original source code.
```

## Introduction

In February 2026 I was modernizing the Python bindings of the Starlink AST library {cite:p}`2016A&C....15...33B` to make it usable as the WCS backend for our file format and data modeling rewrite of the LSST Science Pipeline images {cite:p}`DMTN-339`.
This was needed since there had been a new release of the C library that included critical functionality and we needed those changes [on PyPI](https://pypi.org/project/starlink-pyast/).
For historical reasons (the Python wrapping work started in 2011) the Python interface was written using native C Python APIs and had to support Python 2.6 and 3.x.
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

A port of this size inevitably means that it's impossible to review every line of code and there is a danger that the entire port is effectively "vibe-coded" {cite:p}`10.1080/08874417.2026.2621186,10.48550/arXiv.2506.23253` as more and more code flies by and you always accept each suggestion.
For a small throw-away application this is an acceptable approach, but it comes with risks when porting a large library that is intended for production use.

## The Starlink AST Library

The Starlink AST library is a mature C library that has been in development since 1997 {cite:p}`1998ASPC..145...41W`.
The standard FITS approach for celestial coordinates only defines undistorted coordinates {cite:p}`2002A&A...395.1077C`.
Over the years there have been numerous approaches to support distortions in FITS headers with the ill-fated attempt at a standard {cite:p}`FITSWCSIV` and non-standard proposals such as [TPV](https://fits.gsfc.nasa.gov/registry/tpvwcs/tpv.html) {cite:p}`TPVWCS`, TNX {cite:p}`TNXWCS` and ZPX {cite:p}`ZPXWCS`, with SIP {cite:p}`2005ASPC..347..491S` remaining as the only commonly implemented option, but none of the approaches allow a WCS to be refined and extended in a generalized manner.
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
This graph shows the lines of C code (comments plus code, including tests) as a function of time and includes the main library and associated test code.
It does not include the Fortran test code.
The Git repository started in 1998 after 100,000 lines of code had already been written.
Gaps in the timeline note where at least 3 months had elapsed between commits.
```

## Previous Attempts at Porting the AST Library

In the early 2000s the Starlink Project was instructed to switch away from using C and Fortran with a bespoke message bus (ADAM) and instead switch to the more modern technologies of Java and SOAP {cite:p}`2003ASPC..295..445B`.
As part of that work it was realized that the AST library was a core foundation of the Starlink software ecosystem.
They started work on such a port in early 2002 but by October 2002 they had already decided that as an interim measure there should be a JNI wrapper around the existing AST C library.[^jniast]
This has many disadvantages in that it requires a different binary per supported architecture but it did unblock the use of WCS in tools such as SPLAT {cite:p}`2014A&C.....7..108S`, SoG (Son of GAIA), and Treeview.[^treeview]
By the time of the 2003 ADASS {cite:p}`2004ASPC..314..412B` there was no mention of a native Java port, solely JNI, and the project had been abandoned.

There is no public record of the project's demise, but later discussions (Berry, private communication) indicated that the project involved writing Perl scripts to try to recognize the standard boilerplate in the files and extract the business logic in a way that could be more easily converted to Java.
The project failed in part due to the complexities of mapping C pointer behavior to Java.
There was also some concern that the legacy FITS header support would be particularly challenging to reproduce.

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
The first Rust code was added to the LSST Science Pipelines {cite:p}`PSTN-019` in early 2026.[^rubinoxide]

There are numerous reasons to consider porting the C AST library to Rust.

1. The bespoke object-oriented implementation comes with a lot of boilerplate code and a steep learning curve.
   Rust includes native support for many equivalent concepts, raising the possibility of the same functionality using far less code and being more focused on the algorithms that matter.
2. Memory safety is a core goal of Rust with a very strong ownership and borrowing model making many coding errors relating to memory management and buffer overflows impossible.
3. Rust has a much stronger type system than C, enforced by the compiler, and has explicit approaches for indicating optional return values and error states that can be reasoned about by the compiler and are much easier to track than the AST global status.

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
I had it write benchmarking tooling to compare performance of the polynomial distortions between C and Rust with different numbers of inputs to compare the performance of small numbers of points (C is relatively slow with those) versus large numbers of points (C uses extensive up front set-up that is optimized for large numbers of points).

Towards the end of March there was a significant drop off in code generation as the agent tied itself in knots trying to work on simplification logic and this resulted in a decision to step back from the port and re-assess (once I had run out of tokens in two days).
It also turned out that March 2026 was special in that OpenAI had allocated significantly more tokens to users of the new Codex app than they would subsequently allow towards the end of April, leading to a false sense of abundance that would turn out to be short-lived.

```{figure} starlink-astrs-proto-loc.svg

Lines of Rust code as a function of time for the first Rust prototype.
The gap indicates time not working on the port.
```

### Lessons Learned

```{figure} simplify-passes.svg

An example of how the AST simplify logic can work with multiple passes converting 8 mappings to 3.
Note how it linearizes the mapping chain and then starts walking through from first to last.
Every reduction in mapping count leads to more transform performance.
The simplification engine is one of the places with the most complex logic and the easiest to misunderstand and implement simpler algorithms if the test fixtures are not driving completeness.
```

In about 10 days the port implemented about 15% of the C library's logic, dominated by code for reading a Rubin FITS header and transforming the resultant `FrameSet` from pixel coordinates to sky coordinates and vice versa.
Writing of FITS headers was not implemented but native plain text serialization was added for the supported subset of mappings.

The work did demonstrate one of the key motivations for the investigation.
We have long known that AST is extremely inefficient at transforming small numbers of points.
This is because there are setup costs in the C before a transformation can trigger but those are not necessary in Rust.
For the representative Rubin `FrameSet` a single point transform is 3- to 8x faster than the C.
For large numbers of points in some cases with the `PolyMap` transformation the Rust is 1.2x faster going forward and 1.5x faster for the inverse direction, without using vectorization.
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
   It also created an entirely new class, `TanSkyMap`, specifically to handle one specialist FITS header.
5. The `degen1.ast` example `FrameSet` is one of the trickiest to handle since it has a degenerate WCS with mismatched pixel axes, the Rust test code was failing to generate a simplified form that matched the C and so it had a special clause in the code specifically for this `FrameSet` to reorganize it in the required form.
   The workaround was explicitly turned off when reading similar example files that would be broken by the rewrite.
6. Special-cased handling was also included for some form of CAR headers.

There were also inefficiencies:

1. When parsing a SIP FITS header, the code was creating a serialized native text form of the polynomial definition and handing it to `PolyMap` to be parsed rather than just using the direct `PolyMap` constructor.
2. I wondered about performance issues too early in the process and driving the agent to make something faster led to it putting optimizations in the wrong places which then led to downstream confusion that built on those mistakes.

The fundamental learning from the prototype work was that test coverage of the C library is critical, as is the use of the oracle to provide immediate feedback to compare with the new code when there is a disagreement.
At the time test coverage in AST was mainly handled through Fortran testing with no formal coverage reporting.
This then provided the focus for the next phase.

## Tests Are Important

By April 2026 it was realized that there was some fundamental work that could be done on the C library that could improve the quality of the Rust port.

The first task was to simplify the build system.
The AST C library uses the GNU Autoconf tooling but with Starlink extensions for handling some of the Fortran interfaces {cite:p}`2005ASPC..347..119G`.
The core library is not in itself complicated to build, but the Git source has no means to build it automatically without having access to the Starlink software distribution.
For the prototype we got around this by teaching Codex how to build the oracle based on it working out how the Python bindings were built.
This was not the most efficient approach and so in April 2026 we used Claude Opus 4.6 to build an entire parallel CMake build system that would build the C library (not the Fortran bindings) and run the C tests.
We also methodically converted the Fortran tests to C and added coverage reporting into CI so we could determine the current state of the test suite.
For one of these Fortran test files (about 1300 lines of Fortran) I had run out of Claude tokens and switched to Gemini Pro 3.1 -- it took a couple of hours and didn't make much progress.
When I had access to Claude it looked at the code, determined that the code was a very naive direct Fortran-to-C port, and decided it was easier to rewrite it from scratch, which it did in ten minutes.
Additionally, Gemini had read the instruction that there should be no new warnings introduced so it decided to use `#pragma` to turn off the warnings in its new code.

This work was focused on determining the current code coverage as a whole and not specifically the parts that we wanted to focus on for the initial port, and so included coverage for the plotting subsystem.
Since the Fortran plotting tests required PGPLOT we added a new graphics backend for PLplot and additionally Claude wrote a native SVG plotting backend which was very useful since Claude can read SVG outputs natively and reason about their contents, allowing it to iterate rapidly to ensure that the SVG was rendering correctly.
When this code was merged the code coverage for the core library code was approximately 50%.
Adding GitHub Actions CI also enabled us to easily do sanitizer builds, which immediately found a couple of latent buffer overruns that had been around for a long time.

Once that was in place the next step was to create more example FITS headers.
For this work Claude was tasked with improving the coverage solely of the FITS handling file.
It was able to read the code and then attempt to create a FITS header targeting a specific line.
The FITS header handling is one of the most esoteric parts of the AST source since it encodes many historical variants of FITS headers, many of which are not in wide circulation today, but it was felt that a port that could only read current FITS WCS variants would not be a real port, and we could not abandon the historical support (the AST library is used by the SAOImageDS9 image viewer specifically for this reason).
This work added 90 new FITS headers to the test fixtures, with the code coverage for the FITS handler increasing to 75% -- the remaining coverage was difficult to establish since much of it involves correctness tests that are acting as safety nets confirming a situation that is already handled earlier in the chain.

This work provided the foundation for the next porting attempt.

## Starting Again From Scratch

Once we had a better build and test story for the C library, by late April 2026 we decided to make another attempt at doing the port from scratch.
By this time Claude Opus 4.6 was the current model and in addition to a $20/month team account I had access to a Claude API key through the DOE SLAC National Laboratory near Stanford.
Since Rubin is partially funded by DOE we were allowed to make use of their new infrastructure based on Amazon Bedrock since SLAC was instructed to investigate ways in which LLMs can be used in research activities.
For FY26 SLAC is absorbing the cost of the API keys as part of the research experiment so Rubin was able to make use of this without being charged.

This time we decided to start by planning what we thought the port should look like in terms of memory management and mutability and how traits should relate to C methods.
As was done previously the wcsLib, ERFA, and PAL source code was copied unchanged with Rust shims on top.

Work started on 2026-04-27 with Claude Opus 4.6.
For planning we used the [Superpowers](https://github.com/obra/superpowers) plugin, and this was particularly helpful in iterative specification development and then building and documenting implementation plans.
It defines a full software development process involving using subagents (usually of a lesser model) to implement the plan, and then review of that work by the main agent.
This can give very nice results although subagents tend to use far more tokens than using the main model, even if a lesser model is doing the work.

Work progressed reasonably well through August 2026 with the main impediment to progress being token starvation.
It was possible to hit my monthly SLAC API key ceiling in the first week.
The switch to Opus 5 also helped.
It was noticeable how Fable (for some limited testing) and Opus 5 were able to come to solutions more quickly than the older models, so even though they were nominally more expensive per token the net cost per feature can be less.

```{figure} starlink-astrs-loc.svg

Lines of code as a function of time for the second Rust port.
```

### Lessons Learned

In the prototype we attempted to build a C ABI incrementally in order to ensure that C semantics could work.
On the second attempt we deliberately punted the C ABI until the very last stage once full functionality was ported.
At this time it is unclear whether this was the right approach since it is much harder to change direction after a full implementation.
A simple demonstration of a `FrameSet` and a `ZoomMap` transforming a number from the C side would give more confidence in the approach specified in the spec but nothing has yet been tried.
Even more importantly, the ownership model in Rust where adding a mapping to a `CmpMap` transfers ownership to the `CmpMap` is not how the C behaves where adding a mapping to a `CmpMap` clones the mapping -- once inside the `CmpMap` the mapping is marked as immutable so there is no change in core behavior, but there is a change at the edges where in C if you try to change the state of a mapping after it has been added to a `CmpMap` you get an error but if the Rust port has done a deep copy your modification would look like it is working but would have no effect on the `CmpMap`.

As the port grew in size it again suffered from lack of discipline as interesting side tracks were chased.
This led to the FITS channel handling being implemented before all of the core mappings (especially `TimeFrame`) were complete.
Lacking `TimeFrame` meant that time conversion logic became embedded in the FITS handler rather than in its correct place.

Somewhat embarrassingly this second port made a similar mistake regarding simplification logic and where it was located.
In late May I went back to the AST library and decided to methodically generate hundreds of `CmpMap`s in order to maximize code coverage and provide a true reference corpus of the simplification logic.
This immediately led to vast numbers of failures on the Rust side including a critical discovery that the `CmpMap` flattening in Rust was using a different (but mathematically equivalent) algorithm to the C code -- transforms were working fine before and after simplification but the mappings had a different shape when serialized.
Fixing that took a while to recover with many commits porting C algorithms and reorganizing all the related `MapMerge` logic into each mapping where it should have always resided.
Even by late August with 400 test fixtures passing it was discovered that some parts of the simplification chain were in the wrong place and other places were missing with workarounds.
This was discovered with a full audit comparing the C and Rust side.
It was even discovered that some bespoke logic had been included in the serialization layer to allow a test to pass even though the simplification logic was not fully accurate.
Given that the transform logic was working correctly and the serialization output was incorrect in an edge case, we still needed to match the C logic to give us confidence we had not missed something critical.
Even worse, whilst trying to recover from the earlier mistakes and instrumenting the entire simplification engine we realized that very early on the port had written a `simplified()` API that corresponded to the internal vtable implementation but which incorrectly included some `map_merge` logic when it should not and crucially was called in much of the code instead of the public `simplify()` method.
This was the root cause of the simplification discrepancies since only `simplify()` stamped the `IsSimple` flag and calling the internal method bypassed that flag and led to premature simplifications or redone simplifications.
Furthermore, having some logic in `simplified()` instead of `map_merge` meant that the `CmpMap` simplification could not be reproduced.
Recovering this dominated the late August work.

Because there was no internal tracking of port status in any committed document there were some cases where some classes were added (such as the regions) but only implemented a subset of the internal C logic.
Since nothing was tracking the missing logic there were cases in other parts of the code that worked around the absence rather than understanding that the real fix was to fix it properly.

Early on there were hundreds of test fixtures failing and so those tests were made optional.
This was fine except it was easy to forget that they were failing when all tests were green.
The Rust test infrastructure does not include an "expected fail" state either that could let us know if a fix elsewhere led to a test passing that was not expected.

Many tokens were spent trying to generate the correct invert flags in the `FrameSet` serializations.
These flags record the original state of the invert flag in the mapping since the invert state is owned by the `CmpMap` but because the C clones the mapping it needs to record that original state to make sure that the user's clone does not change.
The Rust ownership model does not need to track this and since the design spec did not realize the significance of these flags in the C code it took many iterations and reverted experiments to get them to match up with the serialization.
This is clearly a C implementation detail but has to be tracked for byte exactness and it took too long to understand its true meaning.

### Porting as Bug Finding

During the porting work there were times when the agent would be unable to reproduce a fixture or a test and determine that there was a possible bug in the upstream C library itself.
As of August 2026 we found numerous bugs and fixed them.
Some of them required updating of reference fixtures and so there was a virtuous cycle of finding a bug, reporting a bug, fixing a bug, merging the upstream fix, and syncing upstream with the port.

The port discovered and led to fixes of the following bugs.
I include them here to give an idea of the different types of issues encountered.
Most of them were very small fixes.

- `RefRA`/`RefDec` were round-tripped through a formatted string at default `Digits`, so spectral `VELOSYS` drifted; now formatted at 20 digits.
- `MatrixMap` `MapSplit` advanced its matrix pointer only past non-BAD elements, so it read uninitialized memory and split results were nondeterministic.
- `ScalePolyInputs`' inverse-coefficient branch indexed off a stale pointer: an out-of-bounds read.
- `DATE-OBS` fractional seconds were parsed with `"%d"`, dropping leading zeros: `.087` became `0.87` s, a 783 ms epoch error.
- `SIPIntWorld` read the output-axis count from `astGetNin` instead of `astGetNout`; a one-word typo.
- `SpecMap`'s velocity helpers cast `RefRA`/`RefDec` to float, discarding ~9 significant digits inside every `LSRK`/`LSRD`/`LG`/`GAL` correction.
- `CLASSFromStore` multiplied the spectral `CDELT` by `specfactor` twice, so FITS-CLASS output carried a squared unit scale.
- `astDat`'s pre-1972 inverse divided by `1.02592` instead of `1 + b/SPD` (off by a seconds-per-day units factor), leaking ~164 ms per UTC→TAI→UTC round trip for 1960s epochs; the same commit fixed a transcribed 1966 intercept (`4.2131700` → `4.3131700`).
- `astAxAngle`'s degenerate-case nudge indexed its coordinate array with the loop variable rather than the axis argument, returning BAD instead of a valid angle.
- `astRate` seeded its numerical step with an absolute 1.0, so it returned BAD at `x0 = 0` for any `Mapping` with a narrow domain (such as `ChebyMap`).
- `astRate`'s zero-range confirmation read the range array at the loop counter instead of the stored index, confirming the wrong axis.
- `PermMap::Equal` skipped comparing the referenced constants when both index entries matched, so `PermMap`s with different constant values compared equal.
- `SpecFluxFrame`'s `astEqual` reported two identical frames as unequal.
- negative grism interference orders were corrupted on read (−2 became −1, −1 became 0) due to negative floats rounding in the wrong direction.
- the `(int)(x+0.5)` idiom rounds every negative value toward zero; fixed in `ConvertValue` and then swept tree-wide across 70 sites in 16 files, including the `WCSAXES` axis count.
- `astMapIterate` never terminated on a `KeyMap` with `SortBy` set: an infinite loop, latent because the only in-tree caller never sets it.
- `astMapRename` on a locked `KeyMap` deleted the entry and then refused to re-add it, so the data was simply lost.
- out-of-range and NaN double-to-integer conversion in `ConvertValue` was undefined behavior; now range-checked with a defined result.
- numeric-string overflow into `%d` was UB, and glibc silently wrapped rather than failing; now detected.
- a zero-length vector entry was dumped by reading an `Entry1X` as an `Entry0X`, writing pointer bytes as the value and null-dereferencing for one type; now rejected at `astMapPut1<X>`.
- `astMapGetElem<X>` reported success on an undefined entry while writing nothing to the caller's buffer, unlike `astMapGet0<X>`/`astMapGet1<X>`.
- `LoadKeyMap` injected a spurious `KyCas = 1` on every re-dump, because `KeyCase`'s unset sentinel is `-1`, not `-INT_MAX`.
- `astLoadKeyMap` applied MpLck before loading any entry, so a locked non-empty `KeyMap` could never be read back at all.
- `LoadKeyMap` read a `Mem%d` card that `DumpEntry` never writes, resetting the member counter so every loaded entry got member 0.
- `astMapCopyEntry` normalized the key with the source `KeyMap`'s `KeyCase`, for both the destination lookup and the stored key.
- `EncodeFloat` stripped a redundant exponent zero by shifting the prefix right rather than closing the gap, so any value wider than 20 columns (e.g. `FitsDigits=17`) was emitted one column right of every other card, running past column 30. Malformed FITS output.
- legacy `CDjjjiii` cards were read (and so consumed) under every encoding but stored only under `FITS-IRAF`, silently losing the rotation matrix elsewhere.
- `SpecTrans` marked only the first copy of a repeated keyword while `WcsFcRead` swept all of them, so reading a two-HDU header left a self-inconsistent partial WCS behind; `MJD-OBS` survival also depended on whether an unrelated `DATE-OBS` sat alongside it.

There were also fixes that enforced stability in the native serialization format.
Whilst the serialization format was historically meant to be treated as a private format understood solely by the C library, validating the Rust port requires that the output is predictable and can be reasoned about.

- `WcsNative` built one `PermMap`, used it in two stages, and inverted it in between; because `astCmpMap` clones rather than copies, the stage-1 `CmpMap` serialized a vestigial `Invert = 1` on its child that the transform path ignores.
- KeyMap dump ordering.
  `Dump` walked hash buckets and `LoadKeyMap` head-inserted on reload, so keys sharing a bucket swapped places on every dump/load cycle and a `KeyMap`'s serialization never converged.
  Now dumped in key order.
- IsSimp tag placement.
  A single `Mapping.flags` bit meant both "dump `IsSimp = 1`" and "do not simplify again", and `astSetInvert` cleared it as a side-effect, in particular as happens when `astEqual` is called.
  Which interior `CmpMap` node ended up carrying the tag therefore depended on shared-pointer aliasing history during simplification, with no structural rule a re-implementation could follow.

Fixes were also made as part of the enhancements to the test coverage, but those are not directly attributable to the Rust port.

### Port Timeline and Status

```{figure} starlink-port-interleave.svg

A timeline for the work described in this document as of mid August 2026.
You can see how the prototype triggered work in the C library to update the build system and add test coverage and that later work on the port resulted in separate work on the C library (bug fixes and additional test fixtures).
The spike in the AST commit history in May is adding many more simplification example test fixtures.
```

The timeline for the port is shown in the figure above and demonstrates the different phases of the project and how the upstream test improvements and bug fixes interleaved with the core porting work.
This timeline includes most of the mappings and all the frames and non-STC regions.
It also includes `Channel` and `FitsChan` but none of the other channels (`YamlChan`, `XmlChan`, `Moc` and `MocChan`, `StcsChan`).
It does not include `Plot`, `Plot3D`, or any `grf` backend, `Table` and `FitsTable`, `IntraMap`, `XphMap`.
Also missing is the core mapping functionality of `astResample`, `astRebin`, `astRebinSeq`, `astTranGrid` and `astQuadApprox`.
These missing features account for about 30% of the code volume of the C library.
There is no C or Python interface and there have been minimal performance optimizations -- the second port is currently slower than the prototype in many cases.

From a distance it looks to the naive user that stacking some transforms together should be relatively straightforward.
Both the ports demonstrate, in their initial velocity, that setting up some core mappings in a `FrameSet` is the quick part.
The complexity comes in all the optimizations and edge cases.
In particular the simplification system is fundamental to core performance -- a user should be able to construct the mappings in a way that is obvious to them and let the library convert them to an optimized but equivalent form.
The frame conversion system is also important and complex.
Porting a library of this size shows that the 80% is relatively straightforward but the final 20% is where all the edge cases and optimizations are.

## Recommendations

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
  Even if you have code coverage for the small area you are optimizing you make it harder later on to reason about whether the Rust version truly matches the C approach and sometimes the optimization can shift logic around that confuses a later code addition that relied on the earlier form.
* You do not want to end up with a line for line port of the source library to the new language.
  You want to ensure that the algorithms are ported (with annotations) whilst making use of new language features.
  For example, for the Rust port we used the standard `Option` and `Result` approach to status handling rather than inherited integer status.
  The AST library is particularly well-suited to a bit since the original authors were very careful to explain all their decisions as detailed comments and to carefully separate each chunk of logic.
  This allowed the Rust port to easily pont at distinct blocks of code in the original.
* Ensure that your coding standards cannot be bypassed.
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
* Some times the agent will flail around trying to work out what is going wrong.
  Remember that you have an oracle.
  Suggest to the agent that it instruments the original library so it can trace exactly what code is involved for a passing test.
  This helped many times in the port of the AST library for mapping simplification verification and for performance validation such as ensuring the same number of trigonometric calls occur in both the original and the port.
  * If the target language has native tracing/debugging/logging facilities embed those features into the port as you go.
    If you do not do this you will notice that the agent is continually adding and removing debugging instrumentation.
    Even better if you can add similar facilities to the original library to allow direct comparison.
    Agents can work much faster if they can turn on trace logic on both sides and immediately see the shape of the difference in logic without having to cycle through phases of code churn each step of the way.

When porting from one language to another you have to decide whether "byte exact" is important or if you are targeting test completeness.
The former is much harder than the latter but is required if this is a mission critical core library.
Once you are byte exact, you are then able to focus on performance optimizations.

[^astshim]: <https://github.com/lsst/astshim>
[^rubinoxide]: <https://github.com/lsst/rubinoxide>
[^jniast]: The JNIAST package in 2002 October included the lines "This package is a temporary measure; in due course it will be superseded by the pure java WCS package.  WCS is being actively worked on now (mid 2002), but the timescale for completion is uncertain."
[^treeview]: The [treeview docs](https://github.com/Starlink/starjava/blob/master/treeview/src/resources/uk/ac/starlink/treeview/docs/homepage.html#L313C5-L313C37) still refer to a todo item of "Pure java version of WCS support" as of August 2026.
[^longterm]: Ironically, the same argument for porting to Rust using coding agents could apply to using those same agents to update the existing C library --- by default the agents will match the coding style of the surrounding code and they are not scared by non-standard inheritance models or clever memory management.

## Conclusions

At the time of writing it is clear that a Large Language Model coding agent such as Claude Opus/Fable or ChatGPT 5.6 Sol, is fully capable of porting large libraries to different languages so long as the original library is well tested.
The test suite, planning, and a reference oracle mitigate many of the problems faced with using LLMs to write large amounts of new code that has no such boundary conditions.
At Rubin the observatory software team is experimenting with using agents to convert LabVIEW to Rust.
The LSST Science Pipelines have 150,000 lines of C++ code in libraries and it is now clear that converting that to Rust is a plausible enterprise if we wish to do so.
Most of those pipelines libraries are significantly less complex than the AST library.

## Acknowledgments

I thank David Berry and Jim Bosch for their feedback on the initial design spec and for answering numerous questions.
In particular David also was tasked with reviewing some of the giant AST pull requests.
I also thank Michael Shehadeh from SLAC IT for his help with the Claude configuration and API key at SLAC.
The project would not have been possible without the generous allocations of tokens from SLAC IT.

## References

```{bibliography}
  :style: lsst_aa
```

## Appendix: Initial Port Design Spec

This is the initial design spec for the overall shape of the port from C as developed iteratively with the Superpowers plugin brainstorming mode on 2026-04-20 with input from David Berry and Jim Bosch.
Some things have changed as the port evolved and an updated spec can be found in the Git history.

```{include} 2026-04-20-rust-port-design.md
```
