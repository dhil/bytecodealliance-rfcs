# Summary

Implement support for the WebAssembly [exception handling
proposal](https://github.com/WebAssembly/exception-handling).

# Motivation
[motivation]: #motivation

The WebAssembly exception handling proposal introduces mechanisms for
raising, catching, and propagating exceptions. The proposal has been
subject to some controversy as the proposal has undergone several
significant revisions, with an early revision being deployed in
production already. As a result there effectively exists two exception
handling proposals at the moment:

* The official W3C WebAssembly exception handling proposal;
* and, the legacy exception handling proposal.

The official proposal has reached consensus, which was backed by a
summarily approval by the CG at the previous in-person meeting. The
proposal is currently in phase 3 with implementations taking shape in
SpiderMonkey and V8. On the toolchain side, the feature is wanted by,
at least, Kotlin and OCaml, though there may be more toolchains
wanting this feature that I am unaware of. Nonetheless, the proposal
is expected to standardised in its current form in a not too distant
future. Thus to remain standards compliant, we will need to eventually
implement the proposal in Wasmtime.

# Requirements
[requirements]: #requirements

* Exception handling support should be a zero-cost feature, i.e. the
  implementation must not affect the run-time performance (execution
  speed, memory usage, etc) of programs that do not use the feature.
* No libunwind (TODO(dhil): recall justification).

# Requirements
[non-requirements]: #non-requirements

* No support for the legacy exception handling proposal.

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
