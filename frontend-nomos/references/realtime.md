# Real-Time & High-Frequency Data

Read when: pushing frequent updates (charts, tickers, streams), using WebSockets/SSE, or racing API calls.

## Rendering ladder (only above ~30 updates/sec per stream)

Below that: RAF-coalesce `setState`, stop. Above: promote stepwise: external store w/ typed ring buffer → split *shape* (React state, rare changes) from *values* (never touch React) → Worker + transferable buffers → SharedArrayBuffer/Atomics → offscreen canvas → decimation (min-max per pixel column, biggest CPU win) → WebGL only if Profiler proves Canvas is bottleneck.

SharedArrayBuffer needs cross-origin isolation (`Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp`). Breaks 3rd-party embeds that don't opt in.

## Things people get wrong

- **Full jitter, not just "backoff".** Equal jitter (`base/2 + rand(0..base/2)`) still herds clients into overlapping windows. Full jitter: `delay = random() * min(cap, base * 2^attempt)`. Reset attempt counter only after connection *held*. Flapping link must not pin everyone at attempt 0.
- **Browsers cannot send WebSocket ping frames.** Half-open TCP connections fail silently: socket stays OPEN, route dead. App-level heartbeat (~15s, 2 missed pongs = close + reconnect) = only real liveness signal.
- **Delivery is at-least-once.** Dedupe by sequence number. After reconnect, request snapshot/reset; don't stitch diverged state.
- **Backpressure is real.** Check `ws.bufferedAmount` before send. State-shaped data: conflate (keep latest, drop stale), don't queue.
- **SSE is underrated for one-way streams.** Server never reads from client? `EventSource` gives HTTP semantics, auth, auto-reconnect free. (Built-in reconnect has no jitter; text-only. At true firehose rates binary WS cheaper. Also consider pushing server-side downsampled buckets, not raw samples.)
- **Generation guards.** Stale callbacks from superseded socket must not corrupt new state. Compare generation id before acting.

## Stale API Responses (search/autocomplete race)

```ts
let activeController: AbortController | null = null;
async function search(q: string) {
  activeController?.abort();
  const c = new AbortController();
  activeController = c;
  try {
    const res = await fetch(`/api/search?q=${q}`, { signal: c.signal });
    if (!res.ok) throw new Error(String(res.status)); // fetch does NOT reject on 404/500
    return await res.json();
  } catch (e) {
    if (e.name === 'AbortError') return; // expected cancellation — not a failure; don't log/metric it
    throw e;
  }
}
```

Subtle part = *reporting*: cancellation must stay invisible in error metrics, else on-call gets paged by user typing.