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
  vulnerabilities and exposures (CVE). **TODO(dhil): Link to evidence for bugs?**

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

# Non-requirements
[non-requirements]: #non-requirements

* No support for the legacy exception handling
  revision. Justification: the legacy revision is being phased out.

* No support for unwinding across host frames. Justification:
  unwinding across host frames would require implementing a full DWARF
  unwinder or equivalent.

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

## CLIF semantics
[clif-semantics]: #clif-semantics

* A new block type `catch` which is a landing pad for exceptions. This
  type of block will have exactly one parameter which will be a
  pointer to the exception value, e.g. in CLIF syntax
```clif
catch block123(v456: i64):
  ...
```
  We disallow regular control flow edges to `catch` blocks. Meaning
  that neither `jump`, `brif`, or `br_table` are allowed to branch
  into a `catch` block. Instead, the control flow edges to `catch`
  must come via a `try_call` instruction.

* A new call instruction `try_call <ok_label>, <exception_label>`,
  reminiscent of LLVM's `invoke`, where `<ok_label>` must be a regular
  block and `<exception_label>` must be `catch` block. The semantics
  is as follows: when `try_call` returns normally, control tranferred
  to the block named by `<ok_label>`; when `try_call` is unwound,
  control is tranferred to the block named by `<exception_label>`
  (note this may happen multiple times in a two-phase exceptions
  scenario).

We do not define a CLIF instruction for throwing an
exception. Instead, exception throwing must be done indirectly via an
imported function (e.g. a Wasmtime builtin libcall implemented in the
host/engine).

We do not hard-wire in support for two-phase exceptions. Though, it
should be possible to encode two-phase exceptions on top of the
proposed constructs. For example, the producer can emit some prelude
code that runs in the beginning of each `catch` to determine whether
the runtime is in the search phase or unwind phase.

## Unwinding across instances
[unwinding-instances]: #unwinding-across-instances

In Wasmtime each instance is equipped with its own vm context
(henceforth `vmctx`). Suppose we call into another instance which
raises an exception, e.g.

```
instance_A -> instance_B -> raise exception E
```

Instance `A` calls into instance `B` which raises an exception
`E`. Let us suppose `A` has installed an exception handler for
`E`. Importantly, we assume the call from A to B has no intermediate
host frames. Now what happens? Instance `B` initiates the unwind,
which is eventually caught by instance `A`. Meanwhile the problem is
that `A` has a different `vmctx` from `B`... **TODO(dhil): on
reflection, I don't understand the problem, why is `B`'s `vmctx` ever
necessary at the handling site? Surely, `A` has sufficient information
to handle `E`?**

## Unwinding across host frames
[unwinding-hosts]: #unwinding-across-host-frames

As stated in the [non-requirements](:non-requirements) section we do
not plan to support unwinding across host frames. However, to
gracefully handle exceptions that attempt to cross wasm-host boundary
we propose to have our host-to-wasm trampoline catch any Wasm-thrown
exceptions and return them via a three-way result type.

Similarly, we will do not plan to supporting raising Wasm exceptions
directly in host code. Instead, we envisage a host function supply an
element of the exception type as an argument, which the wasm-to-host
trampolines will reify as a genuine exception and raise inside the
Wasm code.

For the design of the three-way result type there are multiple options
to consider:

* A genuine three-parameter result type capturing the three ways in
  which a Wasm program can terminate, e.g. `Result<Success, Exception,
  Trap>`.
* A nested result type, e.g. `Result<Result<Success, Exception>,
  Trap>` to remain compatible with Rust's `?` operator.
* Or continue to use `Result<Success, anyhow::Error>` and allow
  callers to try to downcast the `anyhow::Error` to either a trap or
  an exception as needed.

# Open questions
[open-questions]: #open-questions

* We need to decide whether `catch` blocks in CLIF are allowed to use
  values from dominating blocks.

# References
[references]: #references

* [Exception handling in LLVM](https://llvm.org/docs/ExceptionHandling.html)