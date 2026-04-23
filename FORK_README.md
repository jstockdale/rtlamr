# rtlamr — fast-path fork

This is a fork of [bemasher/rtlamr](https://github.com/bemasher/rtlamr)
with one performance patch applied to `r900/r900.go`.

## What's different

**A single change**: `r900.Parse` now early-exits when the main
decoder found no preamble matches in the current block, skipping
the expensive per-block matched filter and quantize passes that
were running unconditionally upstream.

The change:

- Maintains the sliding-window signal buffer on every call (required
  for correctness when packets straddle block boundaries).
- Runs `p.filter()` and `p.quantize()` only when `len(pkts) > 0`.
- Moves `wg.Done()` into a `defer` so both paths release the wait
  group cleanly.

**Measured impact** (r900 package microbenchmarks on Linux amd64,
Go 1.22):

| Benchmark | Upstream | This fork | Change |
| --- | --- | --- | --- |
| `ParseEmpty` (no preamble matches) | ~196 µs/op | ~5.6 µs/op | **~35× faster** |
| `ParseOnePacket` (one match in block) | ~212 µs/op | ~215 µs/op | unchanged |

In a typical deployment, >99% of decode blocks have no r900 preamble
matches (meters transmit briefly and infrequently), so this fast-path
covers the dominant case.

### Why this matters

`r900.Parse` differs from the other protocol Parse functions in that
it runs its own 4-FSK matched filter + quantize on every block, where
scm/scm+/idm/netidm return immediately when `pkts` is empty. Adding
r900bcd to the msgtype list effectively doubles r900's per-block work
because r900bcd wraps r900's Parse via a goroutine.

Under CPU contention (a busy desktop, a multi-decoder scan tool,
several dongles), this pushed per-block processing past the point
where rtlamr could keep up with the 2.4 Msps IQ stream from rtl_tcp,
causing sample drops.

With this patch applied, a six-protocol invocation
(`-msgtype=scm,scm+,idm,netidm,r900,r900bcd`) consumes roughly the
same CPU as the classic four-protocol `-msgtype=all`
(scm, scm+, idm, r900).

## Correctness

`p.filtered` and `p.quantized` are output buffers that carry no
cross-call state — they are fully recomputed from `p.signal` on
every invocation. As long as `p.signal` is kept current (which the
patched code does unconditionally), skipping filter+quantize in the
empty-pkts case is a pure no-op with respect to future decode
output.

The original upstream tests continue to pass. Additional benchmarks
in `r900/r900_bench_test.go` guard against performance regressions
in the fast path.

## Build

Standard Go toolchain (1.19+ works; tested on 1.22):

```
git clone https://github.com/jstockdale/rtlamr
cd rtlamr
go build .
# produces ./rtlamr
```

Or install directly:

```
go install github.com/jstockdale/rtlamr@latest
# installs to $GOPATH/bin/rtlamr (usually ~/go/bin/rtlamr)
```

## Running

Identical to upstream — all flags, message formats, and behavior are
unchanged. See `rtlamr -h` for the flag reference or the upstream
README for usage examples.

## Upstream

Patch file is in this repo as `r900-fastpath.patch`. It's intended
as a drop-in contribution to upstream; the change is small, contained
to one function, and preserves all externally observable behavior.

PR to be opened against bemasher/rtlamr when upstream is active
again.

## License

AGPL-3.0 (same as upstream). All attribution preserved.
