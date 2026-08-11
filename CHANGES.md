# Vendored fork of karlseguin/pg.zig

Upstream: https://github.com/karlseguin/pg.zig
Vendored at commit `b5a1f25acb8d8008c6b15eb68386e16e624c34f7` (ref `master`).

This is a verbatim copy of upstream plus the code patch below and the dependency
divergence noted after it. Drop this vendor copy and re-pin to a tagged upstream
release once upstream supports a timed connection-pool wait on a threaded
`std.Io`.

## Patch: pool-acquire wait works on the threaded `Io` (`src/pool.zig`)

**Symptom.** Under Zig 0.16, `Pool.acquire()` returned `error.ConcurrencyUnavailable`
the instant a caller had to wait for a free connection (pool exhausted). With the
default API pool size (4), live request concurrency above the available connection
count produced intermittent 500s instead of queueing — a regression from 0.15.2,
where acquire blocked until a connection freed.

**Cause.** Upstream bounds the wait by `_timeout` using
`Io.Select.concurrent(Io.sleep, Io.Condition.wait)` — an *async* select combinator.
This project runs the **threaded** `Io` (`Io.Threaded` via `common.globalIo()`,
"Option A, threaded-not-async"), which cannot perform a concurrent select and
returns `error.ConcurrencyUnavailable`.

**Fix.** `Io.Condition` exposes no timed wait, so the exhaustion branch **bounded-polls**:
it drops the mutex, sleeps a short slice (`POOL_ACQUIRE_POLL_NS` ≈ 2 ms, capped by the
remaining `_timeout` budget), re-takes the mutex, and re-checks the predicate. Elapsed
time is summed from the slept slices (no wall-clock `Io` primitive is needed), so the
wait is bounded by the per-acquire `_timeout` and returns `error.Timeout` instead of
blocking forever when a connection is leaked or a query wedges. `release()` still signals
`_cond` (inert while polling), so a real timed wait can be restored verbatim once an `Io`
exposes one. This restores graceful wait-under-load **and** the acquire deadline.
On `Io.sleep` cancellation (server shutdown) the branch re-locks and returns
`error.Timeout` immediately, so in-flight acquirers drain at once instead of
polling out the full `_timeout` — matching the redis pool's `waitForActiveSlot`.

**Trade-off.** Wakeup latency is up to one poll slice (~2 ms) rather than immediate on
`release()`, and the summed-slice clock slightly over-counts (it ignores time spent
re-checking), so the effective deadline is `_timeout` rounded up by at most one slice.
Both are acceptable for a pool acquire; a wedged query is additionally bounded by the
connection-level statement/read timeouts.

Only `Pool.acquire()` and `Conn.peekForError()` (next section) are changed; the
rest of the library is upstream-verbatim.

## Patch: `peekForError` dupes the error before the reader buffer is reused (`src/conn.zig`)

**Symptom.** agentsfleetd's coverage lane (kcov, macOS) panicked
`reached unreachable code` in `proto/Error.zig`'s `parse` whenever a statement
inside an explicit transaction failed (a trigger `RAISE` in a fault-injection
test), killing the whole suite. Fast uninstrumented runs of the same tests
passed.

**Cause.** `Conn.peekForError()` receives a slice borrowed from the reader's
buffer. When the `ErrorResponse` is the last buffered message,
`Reader.buffered()` resets `pos`/`start` to 0 while the returned slice still
points into `buf`. `peekForError` then called `readyForQuery()` *before*
`setErr` duped the payload; when the `ReadyForQuery` frame had not yet arrived,
the socket read inside `readyForQuery` landed at `buf[0..]` and overwrote the
peeked bytes. The first error-field byte became the `Z` frame's
transaction-state byte (`'I'`/`'T'`/`'E'`) — no valid field code — and
`parse`'s exhaustive switch hit `unreachable`. Firing requires the `E` and `Z`
frames to arrive in separate reads, which slow (instrumented) runs hit
reliably and fast runs mostly miss.

**Fix.** Reorder: `setErr(data)` — which dupes the payload into
connection-owned memory — runs before `readyForQuery()`; the stashed
`error.PG` is returned after `readyForQuery()` succeeds. A `readyForQuery`
failure still propagates, as before. Pure ordering change, no trade-off.

## Dependency divergence: `translate-c` is fetched from GitHub, not codeberg

Upstream declares `translate_c` at `git+https://codeberg.org/ziglang/translate-c`.
Codeberg sheds load by refusing anonymous git clones (`remote: Bye`, HTTP 403)
while its web tier stays up, which takes down every downstream Zig build at the
dependency fetch — before a line compiles, and with no retry that can help.

This fork points the same commit (`57c559cf`) at `github.com/ziglang/translate-c`,
which carries it. The change is provably content-neutral: Zig's package hash is
content-addressed and is byte-identical from either forge —
`translate_c-0.0.0-Q_BUWlX1BgCD1wo6uo97prlp9VJ4gxAjwN_vZ7nsSjGN`. If the mirror
ever diverged, the hash would change and the build would refuse it.

Revert this the moment upstream moves off codeberg, or if codeberg becomes the
more reliable host.

## Consuming this fork: tag **and** commit, never the ref alone

agentsfleetd pins this fork by tag *plus* explicit commit:

```
git+https://github.com/agentsfleet/pg.zig.git?ref=v0.0.0-af.3#<commit>
```

The `#<commit>` is load-bearing. `zig fetch` given `?ref=<tag>` **with no
commit** resolved to `52b9f8a` — the upstream base this fork sits on — rather
than the tagged commit. That silently yields plain upstream: no pool-acquire
patch, no `CHANGES.md`, `translate_c` back on codeberg. It reproduced for every
ref tried, tag and branch alike, once the bad resolution was cached; annotated
vs lightweight tags made no difference.

Two consequences worth keeping:

- **Always pin `?ref=<tag>#<commit>`.** `zig fetch --save` writes this form
  itself when handed a commit; hand it one.
- **The package hash is the backstop.** A wrong resolution changes the hash, so
  a mis-pin fails the build rather than shipping a silently unpatched pool. Do
  not paper over an unexpected hash change — check what was actually fetched
  (`grep POOL_ACQUIRE_POLL_NS src/pool.zig` is the quickest tell).

## Tagging

Every commit agentsfleetd consumes gets a tag, `v0.0.0-af.N`, incrementing.
`af.1` is the original pool-acquire patch; `af.2` the 2026-07-24 upstream merge
(tagged retroactively — it shipped untagged); `af.3` the translate-c mirror;
`af.4` the 2026-08-11 upstream merge (`b5a1f25`) plus the `peekForError`
use-after-reset fix.
