# VanishInbox API

The public REST API for [VanishInbox](https://vanishinbox.com) — generate disposable email addresses, receive incoming messages programmatically, and get pushed a webhook the instant mail arrives.

Useful for CI/CD pipelines, automated testing, QA flows, and any workflow that needs a real throwaway inbox without browser interaction.

---

## Base URL

```
https://vanishinbox.com/api/v1
```

All requests must be made over HTTPS.

---

## Authentication

Every request requires a Bearer token in the `Authorization` header.

```
Authorization: Bearer vib_live_your_api_key_here
```

Get your API key from your [VanishInbox dashboard](https://vanishinbox.com/dashboard/keys).

---

## Rate limits

**120 requests per 60-second sliding window**, per API key.

Rate limit state is returned on every response:

| Header                  | Description                              |
| ------------------------ | ---------------------------------------- |
| `X-RateLimit-Limit`     | Maximum requests in the window (120)     |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset`     | Unix timestamp when the window resets    |
| `Retry-After`           | Seconds to wait (only on 429 responses)  |

When you hit the limit you receive a `429 Too Many Requests` response. Back off for the number of seconds in `Retry-After` before retrying. If you're polling for new mail, use `GET /v1/inbox/{address}/wait` (long-poll) or a webhook instead of a tight loop — both are far more efficient against this limit.

---

## Credits

Some endpoints consume credits from your account balance.

| Header                | Description                                   |
| ---------------------- | --------------------------------------------- |
| `X-Credits-Remaining` | Your balance after this request               |
| `X-Credits-Used`      | Credits charged for this request (`0` or `1`) |

When your balance reaches zero, credit-costing requests return `402 Payment Required`. Top up at [vanishinbox.com/dashboard/billing](https://vanishinbox.com/dashboard/billing). Credits never expire.

---

## Endpoints

### `GET /v1/me`

Verify your API key and check your current credit balance. Free — never charges a credit.

**Request**

```
curl https://vanishinbox.com/api/v1/me \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

**Response `200`**

```json
{
  "credits": 450,
  "key": {
    "id": "key_01abc",
    "name": "My test key",
    "prefix": "vib_live_ab"
  }
}
```

---

### `GET /v1/domains`

List the email domains available through the API. Free — never charges a credit.

Use this endpoint to keep your domain list up to date rather than hardcoding it — domains are occasionally added or rotated.

**Request**

```
curl https://vanishinbox.com/api/v1/domains \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

**Response `200`**

```json
{
  "domains": [
    { "name": "fommie.com" },
    { "name": "whoopza.org" },
    { "name": "fommie.online" },
    { "name": "fommie.store" },
    { "name": "whoopza.store" }
  ]
}
```

---

### `POST /v1/inbox/generate`

Generate a new disposable email address. Free — never charges a credit.

**Request**

```
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vib_live_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Body parameters** (all optional)

| Parameter  | Type   | Description                                                                                                                                                                         |
| ---------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `domain`   | string | A specific domain from `/v1/domains`, or `"random"`. Omit for a random pick.                                                                                                        |
| `username` | string | Custom local part. 3–64 characters, lowercase letters/digits/dots/hyphens/underscores, must start and end with a letter or digit. Omit for a generated username like `happycat827`. |

**Example — specific domain**

```
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vib_live_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{ "domain": "fommie.com" }'
```

**Example — custom username**

```
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vib_live_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{ "username": "my-test-user", "domain": "whoopza.org" }'
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "username": "happycat827",
  "domain": "fommie.com",
  "expires_in_seconds": 600,
  "webhook_eligible": true
}
```

> `expires_in_seconds` reflects the inbox TTL (10 minutes). The clock starts from the moment the **first email arrives**, not from when the address is generated.
>
> `webhook_eligible` is `true` only when no custom `username` was supplied — see [Webhooks](#webhooks) below for why.

---

### `GET /v1/inbox/{address}`

Fetch all emails currently in the inbox for a given address. **Costs 1 credit per request.**

Credits are charged before the Redis read. If the read fails, the credit is automatically refunded.

**Request**

```
curl https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "emails": [
    {
      "id": "01abc123",
      "from": "noreply@example.com",
      "subject": "Confirm your email address",
      "text": "Your verification code is 482910",
      "html": "<p>Your verification code is <strong>482910</strong></p>",
      "receivedAt": "2026-06-01T14:22:03.000Z"
    }
  ],
  "expires_in_seconds": 487
}
```

`expires_in_seconds` is `null` when no emails have arrived yet (the inbox key doesn't exist in Redis).

**Empty inbox**

```json
{
  "address": "happycat827@fommie.com",
  "emails": [],
  "expires_in_seconds": null
}
```

---

### `GET /v1/inbox/{address}/wait`

Long-polls for new emails. Returns immediately if emails already exist, otherwise blocks up to the given timeout waiting for one. **Costs 1 credit per request** — same cost as `GET /v1/inbox/{address}`, but far more efficient than tight polling since it only counts once against your rate limit no matter how long it waits.

**Query parameters**

| Parameter | Type    | Description                          |
| --------- | ------- | ------------------------------------- |
| `timeout` | integer | Seconds to wait, max `60`. Default `30`. |

**Request**

```
curl "https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com/wait?timeout=30" \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "emails": [ /* ... */ ],
  "timed_out": false,
  "expires_in_seconds": 580
}
```

---

### `DELETE /v1/inbox/{address}`

Clear all emails from an inbox immediately. Free — never charges a credit.

Idempotent — returns `200` whether or not the inbox existed.

**Request**

```
curl -X DELETE https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "cleared": true
}
```

---

## Webhooks

Instead of polling `GET /v1/inbox/{address}`, register a webhook and VanishInbox will `POST` to your URL the moment an email arrives — typically within a second of delivery.

**Eligibility.** Only addresses generated *without* a custom `username` are webhook-eligible — check the `webhook_eligible` field on the `POST /v1/inbox/generate` response. This is deliberate: custom usernames are free and guessable, so allowing webhooks on them would let anyone pre-claim an address before a real person types the same string into the free web tool at [vanishinbox.com](https://vanishinbox.com). Auto-generated addresses don't have that problem and are always eligible.

**One webhook per address.** Registering a new one replaces any existing subscription. Subscriptions expire with the address (10 minutes), same as the inbox itself.

**Payload.** Deliveries carry event metadata only — sender, subject, timestamp — not the email body. Fetch the body with `GET /v1/inbox/{address}` if you need it; this keeps the payload small and means a compromised webhook URL can't leak message content by itself.

### `POST /v1/inbox/{address}/webhooks`

Registers a webhook for the address. Requires the address to be `webhook_eligible` and owned by the API key making the request. Free.

**Request**

```
curl -X POST https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com/webhooks \
  -H "Authorization: Bearer vib_live_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{ "target_url": "https://your-app.example.com/hooks/vanishinbox" }'
```

`target_url` must be `https`, publicly resolvable, and not point at a private/loopback/link-local address.

**Response `201`**

```json
{
  "id": "9e2b1a3c-...",
  "address": "happycat827@fommie.com",
  "target_url": "https://your-app.example.com/hooks/vanishinbox",
  "secret": "whsec_9f2c...",
  "created_at": "2026-06-01T14:20:00.000Z"
}
```

> `secret` is returned **once**, at registration. Store it — it can't be retrieved again. If you lose it, register a new webhook to get a fresh one.

### `GET /v1/inbox/{address}/webhooks`

Fetches metadata for the registered webhook. The secret is never returned again after creation. Free.

```
curl https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com/webhooks \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

```json
{
  "id": "9e2b1a3c-...",
  "address": "happycat827@fommie.com",
  "target_url": "https://your-app.example.com/hooks/vanishinbox",
  "created_at": "2026-06-01T14:20:00.000Z"
}
```

### `DELETE /v1/inbox/{address}/webhooks`

Removes the webhook subscription for the address. Free.

```
curl -X DELETE https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com/webhooks \
  -H "Authorization: Bearer vib_live_your_api_key_here"
```

```json
{
  "address": "happycat827@fommie.com",
  "deleted": true
}
```

### Verifying a delivery

Every delivery includes a signature header:

```
X-Webhook-Signature: t=1748419260000,v1=<hmac_sha256_hex>
```

Verify it before trusting the payload:

```js
const crypto = require('crypto')

function verify(secret, timestamp, rawBody, signatureHeader) {
  const [, v1] = signatureHeader.split(',')
  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${rawBody}`)
    .digest('hex')
  return expected === v1.split('=')[1]
}
```

`rawBody` must be the exact bytes received — parse the signature first, then `JSON.parse` afterward.

### Example delivery payload

```json
{
  "event": "email.received",
  "address": "happycat827@fommie.com",
  "email_id": "01abc123",
  "from": "noreply@example.com",
  "from_name": "Example App",
  "subject": "Confirm your email address",
  "received_at": "2026-06-01T14:22:03.000Z"
}
```

---

## Error responses

All errors follow the same shape:

```json
{
  "error": {
    "type": "error_type",
    "message": "Human-readable description."
  }
}
```

| Status | Type                   | Description                                                               |
| ------ | ---------------------- | --------------------------------------------------------------------------- |
| `400`  | `invalid_request`      | Missing or invalid parameter (e.g. bad email address, invalid domain)     |
| `401`  | `unauthorized`         | Missing, malformed, or revoked API key                                    |
| `402`  | `insufficient_credits` | No credits remaining                                                      |
| `403`  | `not_eligible`         | Address isn't webhook-eligible, or isn't owned by this API key            |
| `404`  | `not_found`            | No webhook registered for this address                                    |
| `429`  | `rate_limit_exceeded`  | 120 req/min limit hit — check `Retry-After`                               |
| `500`  | `internal_error`       | Server error. Credits are refunded automatically on inbox fetch failures. |

---

## Code examples

### JavaScript / Node.js

```js
const API_KEY = 'vib_live_your_api_key_here';
const BASE    = 'https://vanishinbox.com/api/v1';

async function generateInbox(domain) {
  const res = await fetch(`${BASE}/inbox/generate`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type':  'application/json',
    },
    body: JSON.stringify({ domain: domain ?? 'random' }),
  });
  if (!res.ok) throw new Error(`Generate failed: ${res.status}`);
  return res.json(); // { address, username, domain, expires_in_seconds, webhook_eligible }
}

async function waitForEmail(address, { timeout = 30 } = {}) {
  const res = await fetch(
    `${BASE}/inbox/${encodeURIComponent(address)}/wait?timeout=${timeout}`,
    { headers: { 'Authorization': `Bearer ${API_KEY}` } },
  );
  if (!res.ok) throw new Error(`Wait failed: ${res.status}`);
  const { emails, timed_out } = await res.json();
  if (timed_out) throw new Error(`No email received within ${timeout}s`);
  return emails;
}

// Example: test a registration flow end-to-end
(async () => {
  const inbox = await generateInbox('fommie.com');
  console.log('Using address:', inbox.address);

  // Trigger your email-sending flow here using inbox.address ...

  const emails = await waitForEmail(inbox.address, { timeout: 30 });
  const subject = emails[0].subject;
  const code    = emails[0].text.match(/\d{6}/)?.[0];

  console.log('Subject:', subject);
  console.log('Code:',    code);
})();
```

### Python

```python
import time
import requests

API_KEY = 'vib_live_your_api_key_here'
BASE    = 'https://vanishinbox.com/api/v1'
HEADERS = {'Authorization': f'Bearer {API_KEY}'}

def generate_inbox(domain='random'):
    res = requests.post(
        f'{BASE}/inbox/generate',
        headers=HEADERS,
        json={'domain': domain},
    )
    res.raise_for_status()
    return res.json()  # { address, username, domain, expires_in_seconds, webhook_eligible }

def wait_for_email(address, timeout=30):
    res = requests.get(
        f'{BASE}/inbox/{requests.utils.quote(address, safe="")}/wait',
        headers=HEADERS,
        params={'timeout': timeout},
    )
    res.raise_for_status()
    data = res.json()
    if data.get('timed_out'):
        raise TimeoutError(f'No email received within {timeout}s')
    return data['emails']

# Example: test a registration flow end-to-end
inbox = generate_inbox('fommie.com')
print('Using address:', inbox['address'])

# Trigger your email-sending flow here using inbox['address'] ...

emails = wait_for_email(inbox['address'], timeout=30)
print('Subject:', emails[0]['subject'])

import re
code = re.search(r'\d{6}', emails[0]['text'])
print('Code:', code.group() if code else 'not found')
```

### GitHub Actions (YAML)

Poll for a verification email as part of a CI workflow:

```yaml
- name: Test registration email
  env:
    VANISH_API_KEY: ${{ secrets.VANISH_API_KEY }}
  run: |
    # Generate a fresh inbox
    INBOX=$(curl -sf -X POST https://vanishinbox.com/api/v1/inbox/generate \
      -H "Authorization: Bearer $VANISH_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{}')
    ADDRESS=$(echo $INBOX | jq -r '.address')
    echo "Using address: $ADDRESS"

    # Trigger your registration flow using $ADDRESS here ...

    # Long-poll for the email (up to 30s), one credit either way
    RESULT=$(curl -sf "https://vanishinbox.com/api/v1/inbox/$ADDRESS/wait?timeout=30" \
      -H "Authorization: Bearer $VANISH_API_KEY")
    COUNT=$(echo $RESULT | jq '.emails | length')
    if [ "$COUNT" -gt "0" ]; then
      echo "Email received"
      echo $RESULT | jq '.emails[0].subject'
      exit 0
    fi
    echo "Timed out waiting for email"
    exit 1
```

---

## Tips

**One address per test.** Generate a fresh address for each test case. This prevents emails from different test runs appearing in the same inbox.

**Prefer `/wait` or webhooks over polling.** Both cost the same as a single `GET`, but `/wait` blocks server-side for up to 60 seconds and a webhook pushes the moment mail lands — either beats a `sleep`-and-retry loop on both credits and latency.

**Check `expires_in_seconds`.** If this is `null`, no email has arrived yet. If it's below `60`, the inbox is about to expire — generate a fresh address if you need more time.

**Use `DELETE` between tests.** If you reuse an address across test runs, call `DELETE /v1/inbox/{address}` first to clear any leftover mail from the previous run.

**Domain blocking.** If a site rejects a specific domain, fetch `/v1/domains` and try a different one. All five domains accept inbound mail identically.

**Webhooks need an auto-generated address.** If you need both a predictable address (custom `username`) and a webhook, that combination isn't supported — pick one per use case.

---

## Links

- [VanishInbox](https://vanishinbox.com) — free web inbox, no API key needed
- [Dashboard](https://vanishinbox.com/dashboard) — manage keys and credits
- [Developer docs](https://vanishinbox.com/developers) — full documentation, same content as this README
- [Contact](https://vanishinbox.com/contact) — support and questions
