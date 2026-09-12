# CLAUDE.md - kyte-datastar

## What this is

A Kyte port of the [Datastar](https://data-star.dev) server SDK (after `datastar-go`), shipped as a Kyte
**package**. This is **not a database driver**: it is the server half of a hypermedia framework that drives
the browser from the server over Server-Sent Events. On one long-lived SSE response the server patches DOM
elements, patches signals, runs scripts, and redirects, all as Datastar v1 protocol events
(`datastar-patch-elements` / `datastar-patch-signals`). The public surface is the `Sse` generator; the
builders and the wire encoder underneath are pure and golden-testable. Module/import name: `datastar`
(`import datastar;`, plus `import ds_signals;` to read inbound signals, `import ds_elements;` for options).
See [README.md](README.md) for the full API.

## Build and test

The package is compiled by the installed Kyte toolchain (`~/.kyte/bin/kyte`, built ReleaseFast from the
`kyte` repo); there is no separate build step for the package itself. The suite is entirely offline (no
server, no socket). Run it from the repo root:

```sh
~/.kyte/bin/kyte test                 # wire golden tests + generator round-trip + deltas
```

Tests: `tests/01_wire.ky` pins the exact on-the-wire SSE bytes (the Datastar client parses line by line
and is unforgiving about spelling, ordering, and the trailing blank line); `02_generator.ky` round-trips
the async generator through an in-memory `BufferSink`; `03_deltas.ky` covers the delta builders.

## Working in this repo (how to make a change)

1. **Understand, then plan.** Read the relevant `src/` module and the test that exercises it before
   editing. Keep the change minimal and match the surrounding Kyte style; do not reformat unrelated code.
2. **Verify real behaviour, not just compilation.** "It compiles" is not done. The wire format is a
   byte-exact contract with the browser client, so drive the change through `kyte test` and confirm the
   golden bytes; an offline `BufferSink` round-trip is the closest stand-in for a live stream.
3. **Run the test suite before and after.** `~/.kyte/bin/kyte test` (offline, ASAN-clean is the standing
   bar). Add a golden case for every wire or builder change; the wire tests are the executable spec.
4. **Commit only when asked**, and branch first if you are on `main`. The compiler, kyte-web, and the
   drivers are separate kytelang repos: a change to any of those does not belong here.

## Layout map

The design is SOLID: `Sse` depends on the `SseSink` trait, never a concrete transport, so the live reactor
path and the offline/test path are interchangeable.

- `src/datastar.ky` - `Sse`, the public generator that composes the builders over a sink
  (`Sse.overStream(s)` for a live connection, `Sse.overBuffer(buf)` for tests).
- `src/ds_const.ky` - protocol constants: event names, patch modes, data-line keys, defaults (pure data).
- `src/ds_wire.ky` - the SSE frame encoder, the one place that knows the on-the-wire byte layout (pure).
- `src/ds_elements.ky` - PatchElements / RemoveElement / ExecuteScript line builders + options (pure).
- `src/ds_signals.ky` - the PatchSignals line builder + `readSignalsRaw(req)` inbound reader.
- `src/ds_response.ky` - response helpers. `src/ds_sink.ky` - the transport: the `SseSink` trait,
  `StreamSink` (live), `BufferSink` (tests).
- `tests/` - Kyte test cases (offline golden + generator round-trip).

## Conventions and gotchas

- The SSE frame layout is a strict contract: an `event:` line, optional `id:` / `retry:` lines, a run of
  `data:` lines, then a blank-line terminator. Do not change spacing or ordering without updating the
  golden tests; the client is unforgiving.
- Open follow-ons (per the README): typed `readSignals<T>` awaits the serde binder hook; router
  integration is not wired yet, so use the raw-socket SSE path today; gzip parity and live browser
  verification are still pending.
- Prose follows Indian English with British spellings (behaviour, colour, initialise) and no em dashes;
  never change code identifiers or API names to match.

## Ecosystem context

A Kyte package consumed as a **git-URL dependency** (no central registry). A consumer fetches it with
`kyte get https://github.com/kytelang/kyte-datastar`, which records it in the manifest `project.json` and
pins it in the lockfile `project.lock.json`; the import name is then `datastar`. The compiler that builds
it lives in the `kyte` repo; it pairs with kyte-web for the hypermedia/web story, and the database drivers
(kyte-postgres, kyte-mysql, kyte-mssql, kyte-mongodb) are separate repos.
