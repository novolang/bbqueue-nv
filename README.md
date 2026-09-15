# bbqueue-nv

A single-producer single-consumer queue has exactly one writer and exactly
one reader. This package implements one whose unit of work is a **grant**: a
run of bytes the producer is told the position of, writes into where it
stands, and then makes visible in a single step. It is a port of the Rust
crate [bbqueue](https://github.com/jamesmunns/bbqueue) by James Munns, which
implements the bip-buffer described in
[The Bip Buffer](https://www.codeproject.com/Articles/3479/The-Bip-Buffer-The-Circular-Buffer-with-a-Twist)
by Simon Cooke.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What a grant is

An ordinary queue is written one element at a time. The producer hands a byte
to the queue and the queue puts it away. A **grant** reverses that. The
producer asks for room for the whole frame, and the queue answers where that
room is. The producer writes the frame there, in the caller's own buffer, and
then **commits** the number of bytes the frame turned out to be. The consumer
is told where the readable bytes are, reads them where they are, and
**releases** the ones it is finished with.

Two things follow from that shape, and both matter on a microcontroller. The
frame is written once rather than copied into the queue's own storage. And
the whole frame either fits or does not: the producer asks the question once,
before writing anything.

A grant is invisible until it is committed. Between `grant` and `commit` the
bytes sit in the buffer and are not in the queue, so a consumer that runs in
between sees the queue exactly as it was. The commit changes one index.

A grant is always **contiguous**: one run, never two. A plain ring buffer
that wraps around the end of its storage hands a nine-byte request back as a
seven-byte piece and a two-byte piece, and every caller writes the same
two-part loop. This queue moves the wrap inside itself instead. When the tail
of the buffer is too short for what was asked, the grant is taken from the
front, and the **watermark** records where the written region at the tail
ended. The queue is then **inverted**: the bytes waiting to be read are the
run from `read_at` to the watermark, followed by the run from zero to
`write_at`. A consumer reads those as two grants with a release between them,
and never sees a frame split in half.

The price of that guarantee is stated rather than hidden. A queue can be full
while bytes are free, because free space that is not one run cannot be
granted.

The queue itself holds no bytes. It is four indices and a flag, and the
buffer belongs to the caller. The same queue therefore serves a `Bytes` on a
host, a fixed array in a device's static memory, and a window into a
DMA region, because it does not know which it has.

| Quantity | Value |
| --- | --- |
| Fields in `BbQueue` | 4 integers and 1 boolean |
| Fields in `BbGrant` | 2 integers |
| Producers, consumers | 1 each |
| Bytes the package allocates | 0 |
| Length of a grant that does not fit | 0 |

## Install

```
novo pkg add bbqueue-nv
```

## Example

```novo
use std.bytes
use bbqueue

fn main() [io]
    // The queue holds no bytes. This is the buffer it hands out indices into.
    var buf = bytes.zeros(64)

    // A queue over that buffer. Sixty-four is the caller's promise about its size.
    var q = bbqueue.queue(64)

    // Ask for room for a four-byte frame. `len` is 4, or 0 when there is no room.
    let g = bbqueue.grant(q, 4)
    if g.len > 0
        // Write the frame where the queue said it goes. Nothing is copied.
        for i in 0..g.len
            buf = bytes.set_u8(buf, g.at + i, 0xA0 + i)
        // Make those four bytes visible to the consumer. This is the only visible step.
        q = bbqueue.commit(q, g, 4)

    // The consumer's half: where the readable bytes are, as one contiguous run.
    let r = bbqueue.read(q)
    for i in 0..r.len
        println("${bytes.byte_at(buf, r.at + i) ?? 0}")

    // Give the bytes read back to the producer.
    q = bbqueue.release(q, r, r.len)
    println("${bbqueue.len(q)}")
```

Build and test with `novo pkg build` and
`novo test --isolate tests/bbqueue_tests.nv`. Today `novo test` fails on
purpose: every test reaches a `not implemented: bbqueue.<fn>` panic. The
tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bbqueue` | The whole package: the queue value, the grant value, the producer's `grant`, `grant_max` and `commit`, the consumer's `read` and `release`, and the six functions that report the state. |

## How to choose an entry point

**`grant` is for a producer that knows its length.** It answers a run of
exactly the requested size, or a run of length zero. A frame that does not
fit is not written at all.

**`grant_max` is for a producer that will write whatever fits.** It answers
the largest contiguous run available, which may be of length zero. Use it
when the frame can be truncated or when the producer is draining a source.

**`read` and `release` are the consumer's pair, and they mirror `grant` and
`commit`.** `read` answers the contiguous bytes waiting, and `release` gives
back the ones that have been dealt with.

**`len`, `capacity`, `is_empty`, `is_full`, `watermark` and `is_inverted`
report and change nothing.** `len` is the depth of the queue across the wrap.
The length of the next read is `read(q).len`, which is smaller on an inverted
queue.

## The rules a user needs

1. **The buffer is yours, and the queue never touches it.** `queue(cap)` takes
   a size and not a buffer. `cap` is your promise about the buffer you will
   pass to every read and write you make with this queue, and every index the
   package answers falls inside `0..cap`.
2. **A grant is contiguous or it does not exist.** A partial grant is never
   given. A `len` of zero means there is no room, and `at` means nothing in
   that case. The reference implementation's `grant_exact` has the same rule.
3. **Asking does not reserve.** `grant` and `grant_max` leave the queue
   unchanged, so a producer that asks twice without committing gets the same
   answer twice.
4. **Nothing is visible until `commit`.** A producer may be interrupted
   between the grant and the commit, may take as long as it likes, and may
   decide to write nothing at all.
5. **`commit` and `release` take the grant back, not only a length.** The
   queue has to know where the run was, because a run taken from the front of
   the buffer is the wrap, and the wrap is what moves the watermark.
6. **`used` may be smaller than the grant, and never larger.** Reserving what
   a frame might need and committing what it turned out to be is the point of
   the two steps. A `used` above `g.len` panics, as an index past the end of a
   list does.
7. **A queue can be full with bytes free.** Free space that is split by the
   wrap is not one run, and only one run can be granted. `is_full` answers
   that question, which is why it is a function and not
   `len(q) == capacity(q)`.
8. **A read grant stops at the watermark.** A consumer draining an inverted
   queue reads twice, with a release in between. That is the only place the
   wrap is visible to a caller.
9. **`commit` moves the producer's fields and `release` moves the consumer's,
   and neither moves the other's.** `commit` writes `write_at`, `watermark`
   and `inverted`. `release` writes `read_at`. Every other function writes
   nothing. Two contexts can therefore share a queue by exchanging one field
   each, rather than by locking the whole value.
10. **There is no error type and no failure value.** A zero-length grant is
    backpressure, which is the normal state of a producer ahead of its
    consumer. The reference implementation's `GrantInProgress` has no
    counterpart here, because `commit` is the only thing that produces the
    next queue value.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Here it covers the whole
public surface. Nothing in the package uses `Bytes`, `Str`, `Result` or a
list, so there is no host-only half to leave out.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It holds a thirty-two byte array inside a `@value` struct, runs one
grant, one commit, one read and one release over it, and reports on the
serial port.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

## What is not included

- **The buffer.** A `@value` struct is built whole and replaced whole
  (SPEC section 14.3), so a queue that owned its bytes would copy the whole
  buffer on every byte written. The storage stays with the caller, which is
  also what lets one queue serve a host and a device.
- **A lock, an atomic or a critical section.** Every function is a total
  function from values to values, so there is nothing in this package to
  protect. What two contexts share is the cell they keep the queue value in,
  and that cell belongs to the caller. Rule 9 says which field each side
  publishes.
- **More than one producer or more than one consumer.** The division of
  fields in rule 9 is what makes the queue cheap, and it holds for one writer
  and one reader only.
- **An error type.** See rule 10.
- **Framing.** The queue moves bytes and does not know where one frame ends
  and the next begins. See "Related packages".

## Related packages

- [heapless-nv](https://novo-lang.org/packages/heapless-nv) is the other
  container package for a device with no allocator: a bounded vector, a
  bounded string, a bounded FIFO and a bounded map. Its FIFO moves elements
  by value, one at a time. This queue moves bytes by position, a run at a
  time, and is the one to use when the producer already has the frame
  written somewhere.
- [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv),
  [cobs-nv](https://novo-lang.org/packages/cobs-nv) and
  [frame-nv](https://novo-lang.org/packages/frame-nv) mark where one frame
  ends and the next begins. A transport that drains this queue over a serial
  link needs one of them.
- [deflog-parser](https://novo-lang.org/packages/deflog-parser) and
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) are the
  deferred-logging format on the device side and the host side. A device that
  defers its logging produces the frames this queue is sized for.
- `std.chan` in the standard library is the channel between two tasks on a
  host. It allocates, it blocks, and it carries values rather than bytes.

## Tests

```bash
novo test --isolate tests/bbqueue_tests.nv   # 13 tests
```

The oracle is the upstream `bbqueue` test suite, which covers the wrap, the
watermark, the split-free read and the full-with-bytes-free case. The tests
here hold the buffer as a real `Bytes` and write into it at the index the
queue answers, so each one reads as a caller rather than as index arithmetic.

The suite asserts that a fresh queue grants its whole buffer, that asking
does not reserve, that a grant is contiguous or nothing, that a commit may be
shorter than its grant, that a frame written in place is read in place, that a
reader may release less than it was granted, that the wrap is a watermark and
never a split grant, that a reader never sees a run spanning the wrap, that a
queue can be full with bytes free, that a zero-length grant is backpressure
rather than a failure, that reading an empty queue answers a zero-length
grant, and that `commit` and `release` each move only their own side's fields.

The tests compile today and fail at run, each on the
`not implemented: bbqueue.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bbqueue.BbQueue`, `.BbGrant` | declared |
| `bbqueue.queue` | no |
| `bbqueue.grant`, `.grant_max` | no |
| `bbqueue.commit` | no |
| `bbqueue.read`, `.release` | no |
| `bbqueue.len`, `.capacity`, `.is_empty`, `.is_full` | no |
| `bbqueue.watermark`, `.is_inverted` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
