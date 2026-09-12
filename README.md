# kyte-datastar

A Kyte port of the [Datastar](https://data-star.dev) server SDK
([`datastar-go`](https://github.com/starfederation/datastar-go)) - the hypermedia
framework that drives the browser from the server over **Server-Sent Events**: patch DOM
elements, patch signals, run scripts, redirect, all as SSE events on a long-lived response.

Datastar v1 protocol (`datastar-patch-elements` / `datastar-patch-signals`).

## Design (SOLID)

Each responsibility is its own module rather than one monolithic file:

| Module        | Responsibility                                                                       |
| ------------- | ------------------------------------------------------------------------------------ |
| `ds_const`    | Protocol constants - event names, patch modes, data-line keys, defaults (pure data). |
| `ds_wire`     | The SSE frame encoder - the one place that knows the on-the-wire byte layout (pure). |
| `ds_elements` | PatchElements / RemoveElement / ExecuteScript line builders + options (pure).        |
| `ds_signals`  | PatchSignals line builder + `readSignalsRaw(req)` request reader.                    |
| `ds_sink`     | The SSE transport: `SseSink` trait, `StreamSink` (live), `BufferSink` (tests).     |
| `datastar`    | `Sse` - the public generator that composes the above over a sink.                    |

`Sse` depends on the `SseSink` trait, never a concrete transport (DIP), so the live reactor
path and the offline/test path are interchangeable. The pure builders + encoder make the
exact wire bytes golden-testable with no socket (see `tests/01_wire.ky`).

## Usage

```kyte
import datastar;
import ds_signals;

// inside a raw-socket SSE handler holding an aio.AsyncStream `stream`:
async fn handle(stream: aio.AsyncStream, req: request.Request): void {
    let sse = datastar.Sse.overStream(stream);
    await sse.writeHeaders();                              // text/event-stream preamble

    // read inbound signals (GET/DELETE query `datastar`, else JSON body):
    let signalsJson = ds_signals.readSignalsRaw(req);

    // patch the DOM:
    await sse.patchElementsSimple("<div id=greeting>hello</div>");
    await sse.patchElements("<p>x</p>", ds_elements.PatchElementOptions.inner("#slot"));
    await sse.removeElement("#spinner");

    // patch signals:
    await sse.patchSignals("{\"count\":1}");
    await sse.patchSignalsIfMissing("{\"theme\":\"dark\"}");

    // run script / redirect:
    await sse.executeScript("console.log('done')");
    await sse.redirect("/dashboard");
}
```

## API

`Sse`:

- `Sse.overStream(s: aio.AsyncStream): Sse` - generator over a live connection.
- `Sse.overBuffer(buf: ds_sink.BufferSink): Sse` - generator over an in-memory buffer.
- `async writeHeaders(): int` - SSE HTTP preamble (raw-socket path only).
- `async patchElements(elements, opts)` / `patchElementsSimple(elements)`
- `async removeElement(selector)`
- `async patchSignals(json)` / `patchSignalsIfMissing(json)`
- `async executeScript(src)` / `executeScriptRaw(src, autoRemove)`
- `async redirect(url)`
- `async send(eventType, dataLines, eventId, retryMs)` - low-level escape hatch.

`ds_signals.readSignalsRaw(req): string` - raw inbound signals JSON.

## Status

- Wire format + all line builders: verified byte-exact (golden tests), ASAN-clean.
- Async generator over the sink: round-trip verified offline through `BufferSink`, ASAN-clean.
- **Follow-ons:** typed `readSignals<T>` once the serde binder is exposed as a callable hook;
  router integration so a controller can return an SSE stream (today use the raw-socket path);
  `sse-compression.go` gzip parity; live browser verification against a real Datastar client.

## Test

```sh
# from lang/ (so ../packages resolves)
kyte test kyte-datastar/tests/01_wire.ky
kyte test kyte-datastar/tests/02_generator.ky
```
