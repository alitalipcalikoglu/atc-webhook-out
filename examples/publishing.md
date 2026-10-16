# Publishing events

Your backends publish; they never know who listens.

```bash
curl -s -X POST $WH/v1/events \
  -H "Authorization: Bearer $PUBLISH_KEY" -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-4821-paid" \
  -d '{ "type": "order.paid", "data": { "orderId": 4821, "total": 349.9, "currency": "TRY" } }'
```

`202`:

```json
{ "event": { "id": "evt_mgx1k2ab5d1e…", "type": "order.paid", "data": { … }, "idempotencyKey": "order-4821-paid", "source": "shop-backend", "test": false, "createdAt": "2026-09-17T10:00:01.204Z" }, "deliveries": 3, "duplicate": false }
```

`deliveries` is how many subscribers matched; their deliveries are queued in the same transaction, so once you see `202` nothing can be lost by a crash a millisecond later. An event that nobody subscribes to is still stored (`deliveries: 0`) and can be picked up later by a [replay](replay.md).

## Idempotency

Publishing is often done from a request handler that may be retried. Send an `Idempotency-Key` header (8-255 characters of `A-Z a-z 0-9 . _ : -`, unique per publishing key): a repeat returns `200` with `duplicate: true`, `Idempotent-Replayed: true`, the original event, and queues nothing new. Unlike the 24 h replay window of other POSTs, an event key holds for as long as the event is kept. Keys are scoped to the API key id, so two backends can use the same key text.

## Size and rate

`data` is any JSON value up to `MAX_EVENT_BYTES` (default 64 KiB) when encoded; above that `413 EVENT_TOO_LARGE`. Put ids in the event and let receivers fetch details if payloads grow. `RATE_LIMIT_MAX` requests per key per minute.

## From a backend

```js
export class Webhooks {
  constructor({ url, apiKey, log = console }) { this.url = url.replace(/\/+$/, ''); this.apiKey = apiKey; this.log = log; }

  /** Fire and forget from request handlers; failures are logged, never thrown into the request. */
  publish(type, data, idempotencyKey) {
    return fetch(`${this.url}/v1/events`, {
      method: 'POST',
      headers: { authorization: `Bearer ${this.apiKey}`, 'content-type': 'application/json', ...(idempotencyKey ? { 'idempotency-key': idempotencyKey } : {}) },
      body: JSON.stringify({ type, data }),
      signal: AbortSignal.timeout(5_000),
    }).then((r) => { if (!r.ok) this.log.warn({ status: r.status, type }, 'webhook publish rejected'); })
      .catch((err) => this.log.warn({ err, type }, 'webhook publish failed'));
  }
}
```

Give every backend its own `publish` role key; a leaked key can then only inject events, never read or change subscriptions.

## Ordering

Deliveries to one subscriber are claimed in creation order but run concurrently, so two events published a few milliseconds apart can arrive swapped; retries reorder further. Include a sequence or timestamp in `data` when order matters to the receiver.
