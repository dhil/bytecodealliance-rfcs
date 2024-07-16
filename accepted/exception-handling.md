# Summary

Implement support for the WebAssembly [exception handling
proposal](https://github.com/WebAssembly/exception-handling).

# Motivation
[motivation]: #motivation

The WebAssembly exception handling proposal introduces mechanisms for
raising, catching, and propagating exceptions. The proposal has been
subject to some controversy as the proposal has undergone several
significant revisions, with an early revision being deployed in
production already. As a result there effectively exists two revisions
of exception handling in the wild at the moment:

* The official W3C WebAssembly exception handling revised proposal;
* and, the legacy exception handling revision.

The revised proposal has reached consensus, which was backed by a
summarily approval by the CG at the in-person meeting in Munich in
October 2023. The proposal is currently in phase 3 with
implementations taking shape in SpiderMonkey and V8. On the toolchain
side, the feature is wanted by, at least, Kotlin and OCaml, though
there may be more toolchains wanting this feature that I am unaware
of. Nonetheless, the proposal is expected to standardised in its
current form in a not too distant future. Thus to remain standards
compliant, we will need to eventually implement the proposal in
Wasmtime.

# Requirements
[requirements]: #requirements

* Exception handling support should be a zero-cost feature, i.e. the
  implementation must not affect the run-time performance (execution
  speed, memory usage, etc) of programs that do not use the feature.

* No dependence on `libunwind`. This library provides a C API for
  determining and unwinding call chains in ELF programs. Nonetheless,
  we do not want to bring it into our trusted computing base (TCB),
  because we have found its implementations to be buggy in the past,
  and bugs in libunwind would instantly turn into common
  vulnerabilities and exposures (CVE). TODO(dhil): Link to evidence for bugs?

* Fast unwinding strategy. We require unwinding to be fast enough to
  support [the guest
  profiler](https://docs.wasmtime.dev/examples-profiling-guest.html). Thus,
  it is important that the chosen implementation strategy does not box
  ourselves into a corner. Though, we consider it acceptable if the
  minimal viable product (MVP) of the implementation does not achieve
  peak performance for unwinding as long as there is a viable path to
  improving the performance.

* Compatibility with standard formats. Regardless of unwinding
  strategy, Wasmtime must continue to work with system profilers which
  suport DWARF or frame pointers, e.g. `perf`. Furthermore, in
  Cranelift `cg_clif` should be extended with capabilities to leverage
  the unwind info support to emit appropriate DWARF or Windows
  SEH. Though, we believe a MVP capable of recovering the `vmctx`
  register should suffice for critical use-cases.

* Leveraging unwind info.

# Non-requirements
[non-requirements]: #non-requirements

* No support for the legacy exception handling revision.

# Proposal sketch
[proposal]: #proposal

* Implementation alternatives
  + Side table-based implementation
  + Calling convention-based implementation
  + Bespoke (subset) DWARF unwinder
* Changes to Wasmtime
  + Mapping from PCs to modules for unwinding
* Changes to cranelift
  + Bjorn3's patch?
* Changes to the embedder API
  + Three-way result type (Ok, Exception, Trap)
  + ExceptionRef

# Open questions
[open-questions]: #open-questions

* We must carefully consider interactions between stack unwinding and
  nested host-wasm calls, e.g. `host -> wasm -> host -> wasm`.
