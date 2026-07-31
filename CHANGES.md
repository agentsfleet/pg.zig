# Vendored fork of karlseguin/pg.zig

Upstream: https://github.com/karlseguin/pg.zig
Vendored at commit `52b9f8a59bfd60a3aa75943a14366d859ea11f26` (ref `master`).

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

Only `Pool.acquire()` is changed; the rest of the library is upstream-verbatim.

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
