# RFC: Clock Injection Hook for Fake-Time Testing

## Status

Draft for status-im/nim-chronos. Not yet filed upstream — this document
is the basis for a future RFC issue.

## Motivation

Async code is notoriously difficult to test deterministically. The most
common pattern in production async test suites — sprinkling
`await sleepAsync(...)` and `withTimeout(...)` with generous tolerances
— produces tests that are slow, flaky, and incapable of exercising
edge cases (e.g. "what happens when the timer fires at exactly the
same moment a future resolves?").

The industry-standard fix is to inject the clock. Tokio's
`time::pause()` / `time::advance()` does this for Rust; Java's
`Clock.fixed` / `Clock.offset`; Go's `clockwork` library. In each case
the test installs a deterministic clock source, fires events in
controlled order, and asserts on a fully-synchronous timeline.

chronos already abstracts time through `Moment` and `Duration` (better
than the average async library — most call `getMonoTime()` inline).
The audit work for this RFC (see references) confirms that
`chronos/timer.nim:425-427` (`proc now*(t: typedesc[Moment]): Moment`)
is the *single chokepoint* for every time-dependent behaviour:

- `chronos/internal/asyncengine.nim:126,589,1017` — timer-wheel polling.
- `chronos/internal/asyncfutures.nim:1377,1515,1640` — `sleepAsync`,
  `withTimeout`, deadline tracking.

Every one of these call sites funnels through `Moment.now()`. Making
that call site overridable is sufficient to put the entire timer
infrastructure under a fake clock.

The downstream consumer is `nim-everywhere`, a cross-target Nim
platform library used by IsoNim apps that need backend-neutral async
primitives. Its `FakeAsyncContext` already provides fake-time for code
that routes through its `sleepFor` facade; the chronos hook extends
the same fake clock to chronos's *own* timer wheel, so code that
directly calls `chronos.sleepAsync` / `chronos.withTimeout` /
`chronos.addTimer` is also covered.

## Proposed change

Add a threadvar holding an optional clock source, plus two procs to
install / clear it. Gate the entire mechanism behind
`-d:chronosClockHook` so default builds are byte-identical to the
current upstream.

```diff
+when defined(chronosClockHook):
+  var customMomentSource {.threadvar.}: proc(): Moment {.gcsafe, raises: [].}
+
+  proc setMomentSource*(source: proc(): Moment {.gcsafe, raises: [].}) =
+    ## Override `Moment.now()` for the current thread. Pass `nil` to
+    ## restore the default monotonic-clock implementation. Intended for
+    ## deterministic test infrastructure (e.g. fake-time); production
+    ## builds should leave this unset.
+    customMomentSource = source
+
+  proc clearMomentSource*() {.inline.} =
+    ## Reset the custom moment source on the current thread to `nil`.
+    customMomentSource = nil
+
 proc now*(t: typedesc[Moment]): Moment {.inline.} =
   ## Returns current moment in time as Moment.
+  when defined(chronosClockHook):
+    if customMomentSource != nil:
+      return customMomentSource()
   Moment(value: int64(fastEpochTimeNano()))
```

Total diff: 18 LOC added, 0 removed. Lives entirely in
`chronos/timer.nim`.

## Use case

A fake-time test in nim-everywhere looks like:

```nim
import chronos, nim_everywhere

let ctx = newFakeAsyncContext()
ctx.install()                              # also calls setMomentSource(...)
defer: ctx.uninstall()                     # also calls clearMomentSource()

let slowFut = chronos.sleepAsync(200.milliseconds)
let timed   = chronos.withTimeout(slowFut, 50.milliseconds)

ctx.advance(50)
drainPlatformCallbacks()
check timed.read == false                  # timeout fired, synchronously
```

The `install` proc registers a clock source that reads
`ctx.nowMs * 1_000_000` (ms → ns). chronos's own `processTimers` then
sees the fake clock advance and fires the right callbacks, in the
right order, without any real sleeping.

## Performance impact

In the un-gated build (`-d:chronosClockHook` not set):

- The hook block compiles out entirely. The `when defined(...)` arm
  inside `proc now*` is constant-folded to nothing.
- Resulting assembly is byte-identical to current upstream (verified by
  diffing the generated object files between gated and ungated builds —
  see fork audit log).

In the gated build (`-d:chronosClockHook` set, source NOT installed):

- One threadvar load + one nil-compare + one not-taken branch per
  `Moment.now()` call. On x86-64 with TLS-aware glibc this is ~2-3
  cycles. The audit measured zero observable difference on chronos's
  own test suite running with `-d:chronosClockHook`.

In the gated build with a source installed:

- One threadvar load + one nil-compare + one taken branch + one
  indirect call. Still O(1); cost dominated by the user-supplied
  source proc, not by the dispatch.

`Moment.now()` is called O(callbacks fired) times, not O(events
polled). The overhead is invisible at any practical workload.

## API alternatives considered

**Compile-time clock injection** (e.g. a generic `Clock` type
parameter). Rejected: tests need to install / uninstall the fake
clock dynamically. One test binary may have both fake-time tests and
real-time tests; a `when` switch forces the build to pick one. A
threadvar is per-thread, dynamic, zero cost when unset — strictly
better.

**Module-level mocking** (a `chronos_test` shim that overrides
`Moment.now`). Rejected: chronos's API is used reflectively in many
places (`addTimer`, `withTimeout`, `sleepAsync`, the timer wheel
itself); a shim would have to patch every internal call site. The
proposed hook lives at the single chokepoint they all funnel
through — minimal surface area, maximum coverage.

**Public time-of-day getter** (a `var clockNow: proc()` exposed at
module scope). Rejected: globally-scoped (not per-thread), wouldn't
compose with multi-threaded chronos consumers.

## Backwards compatibility

Zero breakage. The hook is additive:

- New public procs only when `-d:chronosClockHook` is set.
- The `proc now*` signature is unchanged.
- Behaviour with the flag unset is byte-identical to current upstream.
- Behaviour with the flag set but no source installed is observationally
  identical to current upstream (same return value, just a few cycles
  slower).

## References

- Audit: `codetracer-specs/Front-Ends/IsoNim/nim-everywhere-Async-Fork.md`
  § 3 (chronos clock-touching code audit) + § 5 (proposed hook points).
- Downstream consumer: `metacraft-labs/nim-everywhere`
  (https://github.com/metacraft-labs/nim-everywhere) — FakeAsyncContext
  and its `chronos_fake_clock.nim` wiring module.
- Tokio's analogous mechanism:
  https://docs.rs/tokio/latest/tokio/time/fn.pause.html.

## Next steps

After local validation in `metacraft-labs/nim-chronos` (this RFC's
implementation branch — `clock-injection-hook`):

1. File an RFC issue at `github.com/status-im/nim-chronos/issues`
   linking this document.
2. After maintainer sign-off, open a PR with the 18-LOC patch + a
   `tests/testclockhook.nim` exercising `setMomentSource` /
   `clearMomentSource`.
3. Estimated review window: 1-3 months based on Status' historical
   responsiveness on chronos PRs.
