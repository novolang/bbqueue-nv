# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-10-10

The bodies.  Every function the interface published answers, and the
release is out of draft: 0.1.0 over the 0.0.4 interface, with no
signature, type or effect row changed.

- `grant` asks the tail first, and the front only when the tail is too
  short for what was asked.  Answering the larger run instead would wrap
  a producer that had room where it stood, and move the watermark for
  nothing.
- `grant_max` does answer the larger of the two runs, which is the
  question a variable-length frame asks.
- `commit` reads a grant at offset zero as the wrap only when the buffer
  already held bytes.  On an empty queue the run at zero is the tail
  run, and the two are otherwise the same value: reading them alike
  would set a watermark of zero and hide the commit from `read`.
- `release` clears the inversion when the reader reaches the watermark,
  which is what returns it to the front of the buffer.
- A grant taken from the front and committed with nothing written leaves
  the queue holding exactly what it held and records the wrap anyway,
  which is what rule 4's interrupted producer produces.
- The panic a `used` above the grant gets, in `commit` and in `release`,
  carries a plain message rather than an interpolated one.  A heap
  concatenation is refused at the embedded tier, and the probe behind
  the package's device claim compiles this module whole.

`tests/bbqueue_edge_tests.nv` is new, covering what the API suite walks
past: those two panics, the commit of zero bytes on the grant that
wrapped, and `grant_max` in the state where the larger free run is the
one at the front.  Each panic is raised from a helper that builds its
own queue, because a lambda cannot capture a `@value` binding
(SPEC § 14.5).

This release requires novo 0.23.0 or later.

## 0.0.4 — 2026-09-29

README only; no change to the interface.

- "What is not included" no longer says a queue that owned its bytes
  would copy them on every write.  Since the language's `var self`
  methods (SPEC section 14.7) that is no longer so.  The storage stays
  with the caller because a grant is written in place by an interrupt
  handler or a DMA engine.
- "Related packages" points at the language's `Vec[T; N]`,
  `Deque[T; N]`, `String[N]` and `Map[K, V; N]` (SPEC section 14.8) for
  a bounded collection, where it pointed at heapless-nv, which is
  withdrawn.  This queue stays for what the family does not do: a
  contiguous grant that never splits a frame.

## 0.0.3 — 2026-09-25

One test helper changed for novo 0.10.0, where a `Bytes` buffer is
written in place.  The helper that fills a granted region took the
buffer as a plain parameter and wrote through a second name for it.
Under 0.10.0 that write reaches the caller's buffer, and the compiler
cannot see it.  The helper now takes the buffer as a `var` parameter
and writes into it directly, which is what the in-place test means.  No
change to the interface.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-10

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- `BbQueue` and `BbGrant` — four indices and a flag, and a pair of
  indices into a buffer the caller owns.  Both are `@value` structs, so
  a queue lives in an interrupt handler's stack frame or a static cell
  with no header and no allocation.
- `grant` / `commit` and `read` / `release` — the two-step that lets a
  frame be written in place and made visible by one index move.  A grant
  is contiguous or it does not exist, and `commit` takes the grant back
  because the queue has to know where it was: a grant from the front of
  the buffer is the wrap, and the wrap is what moves the watermark.
- `watermark` and `is_inverted`, published so a caller debugging a wrap
  can see the state the guarantee rests on.

**The queue owns no buffer**, and that is the finding this release
records rather than a preference.  A `@value` struct is built whole and
replaced whole (SPEC § 14.3), so a queue that owned its bytes would copy
the whole buffer on every byte written — quadratic in the one operation
the package exists to make cheap.  The published shape keeps the
arithmetic here and the storage with the caller, which is also what lets
one queue serve a `Bytes` on a host and a static region on a device.

**There is no error type**, for two reasons that point the same way.  A
zero-length grant is backpressure rather than a failure; and the
reference implementation's `GrantInProgress` has no counterpart, because
`commit` is the only thing that produces the next queue value and the
type is therefore the check that the Rust API needed a runtime flag for.
The language agrees: a `@value` struct cannot be a `Result` payload
(SPEC § 14.5), so an unboxed outcome's only failure channel would be an
integer code.
