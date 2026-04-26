# Stability & Robustness Fixes

This document tracks proposed and implemented stability fixes to our fork of
`TikTok-Live-Connector`. The original upstream library is no longer
maintained, so we keep our own branch with patches focused on memory safety,
robustness against malformed data, and performance.

## Why this exists

The strint server creates one `TikTokLiveConnection` per `Room`, and rooms
recycle their connection every retry cycle (typically every 2 minutes for
offline / non-existent streamers — see
[server/src/room.ts](../strint/server/src/room.ts)). Over many hours, even a
small per-cycle leak compounds into significant memory growth. We have
observed this in production on a small VM, which is what triggered this
audit.

## Findings

Findings are grouped by category and prioritized. Each item lists the file,
the problem, and the recommended fix. The "verified" items have been
inspected in code; "flagged" items need further investigation before fixing.

### Memory leaks (verified)

#### M1. `setupWebsocket()` leaks the connect-timeout timer on connection failure

**File:** [src/lib/client.ts](src/lib/client.ts) — `setupWebsocket()`

```ts
wsClient.on('open', () => {
    clearTimeout(connectTimeout);     // only cleared on success
    ...
});
wsClient.on('error', (err) => reject(`Websocket connection failed, ${err}`));
...
const connectTimeout = setTimeout(() => reject('Websocket not responding'), 20_000);
```

If the `error` handler fires before `open`, the 20-second timer is never
cleared. The eventual `reject()` is a no-op on the already-settled promise,
but the timer keeps its closure (and the `wsClient` reference it captured)
alive for up to 20 seconds. With reconnect cycles every couple of minutes,
this stacks up.

**Fix:** Clear the timer in the error handler too.

```ts
const cleanup = () => clearTimeout(connectTimeout);
wsClient.on('open', () => {
    cleanup();
    ...
});
wsClient.on('error', (err) => {
    cleanup();
    reject(`Websocket connection failed, ${err}`);
});
const connectTimeout = setTimeout(() => {
    cleanup();
    reject('Websocket not responding');
}, 20_000);
```

#### M2. Duplicate `'error'` listener after `open`

**File:** [src/lib/client.ts](src/lib/client.ts) — `setupWebsocket()`

The pre-`open` `error → reject` handler is never removed. Once `open` fires,
a second `error → handleError` handler is added. Both stay attached for the
life of `wsClient`. The first one calls `reject` on the already-settled
promise on every subsequent error — functionally a no-op, but it's a stale
listener and a closure pinning `wsUrl`/`wsParams` strings.

**Fix:** Use a named handler and remove it on `open`.

```ts
const onPreOpenError = (err: unknown) => {
    cleanup();
    reject(`Websocket connection failed, ${err}`);
};
wsClient.on('error', onPreOpenError);
wsClient.on('open', () => {
    cleanup();
    wsClient.off('error', onPreOpenError);
    wsClient.on('error', (e) => this.handleError(e, 'WebSocket Error'));
    ...
});
```

#### M3. `disconnect()` does not tear the wsClient down

**File:** [src/lib/client.ts](src/lib/client.ts) — `disconnect()`

```ts
async disconnect(): Promise<void> {
    this.waitUntilLiveAbortController?.abort();
    this.waitUntilLiveAbortController = null;
    if (this.isConnected) {
        this.wsClient?.close();
    }
}
```

Three problems:

1. `this.wsClient` is never set back to `null`. The
   `TikTokLiveConnection` retains a strong reference to a closed socket.
2. No `removeAllListeners()`. Six listeners
   (`open`, `close`, two `error`, `protoMessageFetchResult`, `imEnteredRoom`,
   `webSocketData`, `messageDecodingFailed`) remain attached after close.
   They hold `this` via `processProtoMessageFetchResult.bind(this)`.
3. `close()` is async. Events can still fire on the closing socket and
   re-emit through `this` after the caller assumed disconnect was complete.

**Fix:**

```ts
async disconnect(): Promise<void> {
    this.waitUntilLiveAbortController?.abort();
    this.waitUntilLiveAbortController = null;

    const ws = this.wsClient;
    if (ws) {
        ws.removeAllListeners('protoMessageFetchResult');
        ws.removeAllListeners('imEnteredRoom');
        ws.removeAllListeners('webSocketData');
        ws.removeAllListeners('messageDecodingFailed');
        ws.removeAllListeners('error');
        ws.removeAllListeners('close');
        ws.close();
        this.wsClient = null;
    }
    this.setDisconnected();
}
```

> **Note:** `TikTokWsClient` extends `ws.WebSocket`. Calling
> `removeAllListeners()` without arguments would also drop the internal
> `'message'` and `'close'` handlers registered in the `TikTokWsClient`
> constructor. Either remove specific listener names (as above) or have
> the client expose a dedicated `dispose()` method.

#### M4. `setupWebsocket` is not abortable

**File:** [src/lib/client.ts](src/lib/client.ts) — `setupWebsocket()`

`waitUntilLive` honors the abort controller, but `setupWebsocket`'s 20s
timeout and the in-flight WebSocket handshake are not. Calling `disconnect()`
during the handshake does not cancel anything — the timer keeps the closure
alive until it fires.

**Fix:** Pass the abort signal into `setupWebsocket` and:

- abort the underlying socket via `wsClient.terminate()` if signal aborts
- clear the timer on abort

### Memory leaks (flagged, lower confidence)

#### M5. CookieJar axios interceptors

**File:** [src/lib/web/lib/cookie-jar.ts](src/lib/web/lib/cookie-jar.ts)

The constructor registers axios request/response interceptors that are never
ejected. Each `WebcastHttpClient` creates a fresh axios instance, so the
interceptors die with the instance — **not** a leak in the current usage
pattern (one client per connection). It would only matter if axios instances
are shared. **Recommendation: leave as-is unless usage changes.**

### Security

#### S1. No size cap on protobuf decode

**File:** [src/lib/utilities.ts](src/lib/utilities.ts) — `deserializeMessage()`

```ts
deserializedMessage = messageFn.decode(binaryMessage);
```

`deserializeMessage` blindly decodes `binaryMessage` regardless of size. The
outer `ws` socket's `maxPayload` (default 100 MB) is the only bound. Inside
`ProtoMessageFetchResult`, the `messages` array is iterated without a count
cap, and each nested decode allocates objects.

This is currently low-impact because we only connect to TikTok itself, but
it's a bug we should patch:

**Fix:**

1. Pass `maxPayload` to the underlying `ws.WebSocket` constructor (e.g.
   4 MB — TikTok messages are well under this).
2. In `deserializeMessage`, when handling `ProtoMessageFetchResult`, break
   out of the inner loop after a sane limit (e.g. 5000 nested messages).

```ts
const MAX_NESTED_MESSAGES = 5000;
if (protoName === 'ProtoMessageFetchResult') {
    const msgs = (deserializedMessage as ProtoMessageFetchResult).messages || [];
    if (msgs.length > MAX_NESTED_MESSAGES) {
        msgs.length = MAX_NESTED_MESSAGES;
    }
    for (const message of msgs) {
        ...
    }
}
```

#### S2. Hardcoded sign API key (in strint, not in this lib)

**File:** [server/src/room.ts](../strint/server/src/room.ts)

`EULER_SIGN_API_KEY` is a hardcoded constant. Move it to an env var
(`EULER_SIGN_API_KEY=...`) so it can be rotated without a code change.
Tracked here for completeness; fix lives in the strint repo.

#### S3. Localhost cookie leak (low severity)

**File:** [src/lib/web/lib/http-client.ts](src/lib/web/lib/http-client.ts)

The non-secure-host detection (`127.0.0.1`, `localhost`, `::1`) means session
cookies could be sent in plaintext over HTTP if a developer misconfigures a
proxy. Real traffic only goes to TikTok hosts, so leave for now. Worth
revisiting if we ever introduce a debug proxy.

### Optimizations

#### O1. O(n) gift lookup per gift message

**File:** [src/lib/client.ts](src/lib/client.ts) — gift handler in
`processDecodedData`

```ts
data.extendedGiftInfo = this.availableGifts.find((x) => x.id === data.giftId);
```

`availableGifts` contains thousands of entries. On a high-traffic stream
this is a hot path. Build a `Map<id, gift>` once when `fetchAvailableGifts`
returns, and use `.get()`.

```ts
private _giftMap: Map<number, GiftInfo> | null = null;

public async fetchAvailableGifts() {
    const gifts = await ...;
    this._availableGifts = gifts;
    this._giftMap = new Map(gifts.map(g => [g.id, g]));
    return gifts;
}

// in processDecodedData:
data.extendedGiftInfo = this._giftMap?.get(data.giftId);
```

#### O2. `processProtoMessageFetchResult.bind(this)` allocated per `setupWebsocket`

**File:** [src/lib/client.ts](src/lib/client.ts) — `setupWebsocket()`

A new bound function is created on every connect. Bind once in the
constructor:

```ts
constructor(...) {
    super();
    this.processProtoMessageFetchResult = this.processProtoMessageFetchResult.bind(this);
    ...
}
```

Trivial allocation, but it also keeps the listener identity stable, which
makes `removeListener` calls (see M3) easier.

## Implementation priority

The likely cause of the gradual memory growth observed in strint is the
combination of **M1 + M3 + M4**: every retry cycle (about every 2 minutes for
a non-existent or offline streamer) creates a new `TikTokLiveConnection` and
disconnects it imperfectly. Each cycle leaks a small amount until the VM
runs out of memory.

Recommended order:

1. **M3** — `disconnect()` cleanup. Highest impact.
2. **M1** + **M2** — connect-timeout and pre-open error listener cleanup.
3. **M4** — make the handshake abortable.
4. **S1** — protobuf size/count caps. Cheap defense in depth.
5. **O1** — gift map. Easy CPU win on busy streams.
6. **O2** — pre-bound listener.

## Testing notes

After implementing the fixes, we should be able to verify with a soak test:

- Start a strint server on a small VM with a heap snapshot at boot.
- Subscribe to several non-existent usernames so rooms cycle through retries
  rapidly.
- Take heap snapshots at 1h, 4h, 12h.
- Compare retained-size totals for `TikTokLiveConnection`, `TikTokWsClient`,
  and `Timeout` objects. The fix is successful when these counts stay flat
  rather than growing.

To force more aggressive cycling for testing, temporarily lower
`WAIT_UNTIL_LIVE_RETRY_DELAY_MS` and `CONNECT_RETRY_DELAY_MS` in
[server/src/room.ts](../strint/server/src/room.ts).

## Status

| ID  | Title                                          | Status      |
|-----|------------------------------------------------|-------------|
| M1  | connect-timeout cleanup                        | not started |
| M2  | duplicate error listener                       | not started |
| M3  | disconnect tears down wsClient                 | not started |
| M4  | abortable handshake                            | not started |
| M5  | cookie-jar interceptors (flagged)              | not planned |
| S1  | protobuf size/count caps                       | not started |
| S2  | hardcoded sign API key (strint repo)           | not started |
| S3  | localhost cookie leak (flagged)                | not planned |
| O1  | gift Map for O(1) lookup                       | not started |
| O2  | pre-bound `processProtoMessageFetchResult`     | not started |
