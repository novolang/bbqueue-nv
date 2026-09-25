# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
