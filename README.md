# bbqueue-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A single-producer single-consumer byte queue whose unit of work is a
**contiguous grant**.  The producer asks for N bytes, is told where they
are, writes them where they are, and commits M of the N.  The consumer
is told where the readable bytes are, reads them where they are, and
releases K of them.  The wrap around the end of the buffer is handled by
a **watermark**, never by splitting a grant in two.

It is the buffer under a deferred-logging transport: interrupts produce
frames, one drainer moves them to RTT or a UART, and nothing copies.

## Adding it, and checking it

```bash
novo pkg add bbqueue-nv      # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/bbqueue_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: bbqueue.<fn>`.  They turn green
one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use bbqueue

// The producer's half: ask for room, write the frame where the room is,
// commit what the frame turned out to be.
fn main() [io]
    var buf = bytes.zeros(64)
    var q = bbqueue.queue(64)

    let g = bbqueue.grant(q, 16)
    if g.len > 0
        buf = bytes.set_u8(buf, g.at, 0xAB)
        q = bbqueue.commit(q, g, 1)

    println("${bbqueue.len(q)}")   // 1
```

## Why a byte queue for an interrupt producer wants grants

A push-and-pop queue asks the producer for **a byte at a time, by
value**.  An interrupt handler with a nine-byte log frame in hand
therefore does three things it should not have to:

1. **It copies.**  The frame is already somewhere — in a register file,
   in a struct on the interrupt stack — and every `push` moves one byte
   of it into the queue's own storage.  A grant makes the queue's
   storage the place the frame is assembled in the first place, so the
   frame is written once instead of twice.
2. **It cannot fail atomically.**  Nine pushes are nine chances to run
   out of room, and a producer that discovers it on the seventh has
   already put six bytes of a corrupt frame where a consumer can read
   them.  A grant asks the whole question once: either there is room for
   the frame or there is not, and nothing is visible until `commit`.
3. **It pays per byte.**  Every push is a bounds check, an index
   increment and — across an interrupt boundary — a synchronisation.  A
   grant pays that once for the frame.

**The release is what makes it visible**, and that is the third property
and the load-bearing one.  Between `grant` and `commit` the bytes are in
the buffer and are not in the queue: a consumer that runs in the middle
sees the queue exactly as it was.  So a producer can be interrupted, can
take as long as it likes, and can even decide to write nothing — the
commit is the only instant at which anything changes, and it changes one
index.

And the **contiguity** is what makes all of that usable.  A ring buffer
that wraps hands a nine-byte frame back as a seven-byte piece and a
two-byte piece, and every caller then writes the same two-part loop.
This one moves the wrap into the queue: when the tail is too short, the
grant comes from the front and the watermark records where the tail run
ended.  A caller of this package never handles half a frame.  The price
is stated rather than hidden — `tests/bbqueue_tests.nv` asserts it — **a
queue can be full with bytes free**, because free space that is not one
run cannot be granted.

## The layer, and why

`core`.  Every function is total, from values to values: the queue is
four indices and a flag, it owns no bytes, and it neither reads nor
writes one.  There is no effect to declare because there is nothing here
to perform.

`tests/embedded_probe.nv` is the device claim in a form that either
builds or does not, and it takes the **whole** public surface rather
than a core subset of it — nothing here touches `Bytes`, `Str` or
`Result`, so there is no host-only half to leave out.

**The queue owns no buffer, and that is the finding of this lane rather
than a preference.**  A `@value` struct is built whole and replaced
whole (SPEC § 14.3): there is no field assignment, not on a binding, not
on a nested field, and not into a `[T; N]` element.  So a queue that
owned its bytes would copy the entire buffer on every byte a producer
wrote — O(n²) for the one operation the package exists to make cheap.
The honest shape is the one published: the queue is the arithmetic, the
buffer is the caller's, and a grant is a pair of indices into it.

That is also the better `core` design, and it is why the interface is
not apologetic about it: the same queue serves a `Bytes` on a host, a
static region on a device and a memory-mapped window into a DMA buffer,
because it has no idea which it is.  What the language would need for
the other shape is an unboxed aggregate that can be updated in place —
a mutable `@value` binding, or a borrow of one field of one — and the
lane's report names it.

## The load-bearing interface

Two `@value` structs and the fact that `commit` takes the grant back.

```novo
pub @value
struct BbQueue
    cap: Int
    write_at: Int
    read_at: Int
    watermark: Int
    inverted: Bool

pub @value
struct BbGrant
    at: Int
    len: Int

pub fn grant(q: BbQueue, want: Int) -> BbGrant
pub fn commit(q: BbQueue, g: BbGrant, used: Int) -> BbQueue
pub fn read(q: BbQueue) -> BbGrant
pub fn release(q: BbQueue, g: BbGrant, used: Int) -> BbQueue
```

`commit` takes `g` and not only `used` because the queue has to know
**where** the grant was: a grant taken from the front of the buffer is
the wrap, and the wrap is what moves the watermark.  That one parameter
is what keeps `BbQueue` down to five fields and what makes the grant a
value with a job rather than a return type.

**There is no error type in this package.**  Two reasons, and they point
the same way.  A zero-length grant is backpressure, not a failure — a
producer ahead of its consumer is the normal state of a log queue, and a
caller would write the same `if` either way.  And the reference
implementation's other error, `GrantInProgress`, has no counterpart
here: a Rust grant borrows the queue mutably and the runtime has to
check that a second one is not taken, while here `commit` is the only
thing that produces the next queue value, so the type is the check.

The language pushes the same way.  A `@value` struct cannot be a
`Result` payload, an optional payload or a field of a boxed struct
(SPEC § 14.5), so a function answering `Result<BbQueue, BbFault>` would
not compile at all, and the only failure channel an unboxed outcome has
is an integer code.  Here that costs nothing, because there was nothing
to report; a package where it costs something should say so.

## The critical section, and why it is around one field

Every function here is a total function on values, so **there is nothing
in this package to protect**.  What two contexts share is not the queue
— it is the *cell* they keep the queue value in, and that cell belongs
to the caller.

The division that makes the cell cheap to protect is declared and
tested:

* `commit` moves `write_at`, `watermark` and `inverted`, and nothing
  else.
* `release` moves `read_at`, and nothing else.
* `grant`, `read`, `len` and the rest move nothing.

So an interrupt producer and a main-loop consumer can each hold their
own copy of the queue value and exchange only the field the other side
owns.  That is exactly what the reference implementation's two atomics
carry, and it is why the section a novo-lang port needs is around **one
word** rather than around the queue: the producer publishes `write_at`,
the consumer publishes `read_at`, and neither ever writes the other's.

`critical-section-nv` (planned, `mono`) is where that acquire/release
contract will live — one implementation per platform, PRIMASK on a
Cortex-M and the runtime's section on a host — and every transport and
encoder in the deferred-logging story shares it.  This package
deliberately does **not** declare a trait for it and does **not** carry
a `[hw]` row: a queue with no shared state that declared a lock would be
inventing a need, and an effect row on a function that performs nothing
is a claim the compiler would have to be lied to about.  When the
section exists, it wraps the caller's cell and this package is unchanged.

## The reference implementation

`bbqueue` (MIT/Apache-2.0, James Munns), whose `BBBuffer`, `GrantW`,
`GrantR` and watermark rule this ports.  Three things change, and each
is named where it happens:

* the grant is an **index pair** rather than a mutable slice borrowed
  from the queue, because a `@value` struct cannot be borrowed a field
  at a time;
* `GrantInProgress` is gone, because value threading is what the Rust
  API needed the flag to enforce;
* the buffer is the **caller's** rather than the queue's, for the
  immutability reason above.

The upstream test suite is the oracle for everything else: the wrap, the
watermark, the split-free read, and the "full with bytes free" case
`tests/bbqueue_tests.nv` asserts.

## Status

| function | implemented |
| --- | --- |
| `bbqueue.queue` | no |
| `bbqueue.grant`, `.grant_max` | no |
| `bbqueue.commit` | no |
| `bbqueue.read`, `.release` | no |
| `bbqueue.len`, `.capacity`, `.is_empty`, `.is_full` | no |
| `bbqueue.watermark`, `.is_inverted` | no |
