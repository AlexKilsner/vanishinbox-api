# VanishInbox API

The public REST API for [VanishInbox](https://vanishinbox.com) — generate disposable email addresses and poll for incoming messages programmatically.

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
Authorization: Bearer vi_your_api_key_here
```

Get your API key from your [VanishInbox dashboard](https://vanishinbox.com/dashboard).

---

## Rate Limits

**120 requests per 60-second sliding window**, per API key.

Rate limit state is returned on every response:

| Header | Description |
|---|---|
| `X-RateLimit-Limit` | Maximum requests in the window (120) |
| `X-RateLimit-Remaining` | Requests remaining in the current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait (only on 429 responses) |

When you hit the limit you receive a `429 Too Many Requests` response. Back off for the number of seconds in `Retry-After` before retrying.

---

## Credits

Some endpoints consume credits from your account balance.

| Header | Description |
|---|---|
| `X-Credits-Remaining` | Your balance after this request |
| `X-Credits-Used` | Credits charged for this request (`0` or `1`) |

When your balance reaches zero, inbox fetch requests return `402 Payment Required`. Top up at [vanishinbox.com/dashboard/billing](https://vanishinbox.com/dashboard/billing).

---

## Endpoints

### `GET /v1/me`

Verify your API key and check your current credit balance. Free — never charges a credit.

**Request**

```bash
curl https://vanishinbox.com/api/v1/me \
  -H "Authorization: Bearer vi_your_api_key_here"
```

**Response `200`**

```json
{
  "credits": 450,
  "key": {
    "id": "key_01abc",
    "name": "My test key",
    "prefix": "vi_ab"
  }
}
```

---

### `GET /v1/domains`

List the email domains available through the API. Free — never charges a credit.

Use this endpoint to keep your domain list up to date rather than hardcoding it — domains are occasionally added or rotated.

**Request**

```bash
curl https://vanishinbox.com/api/v1/domains \
  -H "Authorization: Bearer vi_your_api_key_here"
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

```bash
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vi_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Body parameters** (all optional)

| Parameter | Type | Description |
|---|---|---|
| `domain` | string | A specific domain from `/v1/domains`, or `"random"`. Omit for a random pick. |
| `username` | string | Custom local part. 3–64 characters, lowercase letters/digits/dots/hyphens/underscores, must start and end with a letter or digit. Omit for a generated username like `happycat827`. |

**Example — specific domain**

```bash
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vi_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{ "domain": "fommie.com" }'
```

**Example — custom username**

```bash
curl -X POST https://vanishinbox.com/api/v1/inbox/generate \
  -H "Authorization: Bearer vi_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{ "username": "my-test-user", "domain": "whoopza.org" }'
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "username": "happycat827",
  "domain": "fommie.com",
  "expires_in_seconds": 600
}
```

> The `expires_in_seconds` value reflects the inbox TTL (10 minutes). The clock starts from the moment the **first email arrives**, not from when the address is generated.

---

### `GET /v1/inbox/{address}`

Fetch all emails currently in the inbox for a given address. **Costs 1 credit per request.**

Credits are charged before the Redis read. If the read fails, the credit is automatically refunded.

**Request**

```bash
curl https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com \
  -H "Authorization: Bearer vi_your_api_key_here"
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
      "receivedAt": "2025-06-01T14:22:03.000Z"
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

### `DELETE /v1/inbox/{address}`

Clear all emails from an inbox immediately. Free — never charges a credit.

Idempotent — returns `200` whether or not the inbox existed.

**Request**

```bash
curl -X DELETE https://vanishinbox.com/api/v1/inbox/happycat827@fommie.com \
  -H "Authorization: Bearer vi_your_api_key_here"
```

**Response `200`**

```json
{
  "address": "happycat827@fommie.com",
  "cleared": true
}
```

---

## Error Responses

All errors follow the same shape:

```json
{
  "error": {
    "type": "error_type",
    "message": "Human-readable description."
  }
}
```

| Status | Type | Description |
|---|---|---|
| `400` | `invalid_request` | Missing or invalid parameter (e.g. bad email address, invalid domain) |
| `401` | `unauthorized` | Missing, malformed, or revoked API key |
| `402` | `insufficient_credits` | No credits remaining |
| `429` | `rate_limit_exceeded` | 120 req/min limit hit — check `Retry-After` |
| `500` | `internal_error` | Server error. Credits are refunded automatically on inbox fetch failures. |

---

## Code Examples

### JavaScript / Node.js

```javascript
const API_KEY = 'vi_your_api_key_here';
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
  return res.json(); // { address, username, domain, expires_in_seconds }
}

async function pollInbox(address, { intervalMs = 3000, timeoutMs = 60000 } = {}) {
  const deadline = Date.now() + timeoutMs;

  while (Date.now() < deadline) {
    const res = await fetch(`${BASE}/inbox/${encodeURIComponent(address)}`, {
      headers: { 'Authorization': `Bearer ${API_KEY}` },
    });

    if (!res.ok) throw new Error(`Inbox fetch failed: ${res.status}`);

    const { emails } = await res.json();
    if (emails.length > 0) return emails;

    await new Promise(r => setTimeout(r, intervalMs));
  }

  throw new Error(`No email received within ${timeoutMs / 1000}s`);
}

// Example: test a registration flow end-to-end
(async () => {
  const inbox = await generateInbox('fommie.com');
  console.log('Using address:', inbox.address);

  // Trigger your email-sending flow here using inbox.address ...

  const emails = await pollInbox(inbox.address, { intervalMs: 3000, timeoutMs: 60000 });
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

API_KEY = 'vi_your_api_key_here'
BASE    = 'https://vanishinbox.com/api/v1'
HEADERS = {'Authorization': f'Bearer {API_KEY}'}


def generate_inbox(domain='random'):
    res = requests.post(
        f'{BASE}/inbox/generate',
        headers=HEADERS,
        json={'domain': domain},
    )
    res.raise_for_status()
    return res.json()  # { address, username, domain, expires_in_seconds }


def poll_inbox(address, interval=3, timeout=60):
    deadline = time.time() + timeout
    while time.time() < deadline:
        res = requests.get(
            f'{BASE}/inbox/{requests.utils.quote(address, safe="")}',
            headers=HEADERS,
        )
        res.raise_for_status()
        emails = res.json().get('emails', [])
        if emails:
            return emails
        time.sleep(interval)
    raise TimeoutError(f'No email received within {timeout}s')


# Example: test a registration flow end-to-end
inbox = generate_inbox('fommie.com')
print('Using address:', inbox['address'])

# Trigger your email-sending flow here using inbox['address'] ...

emails = poll_inbox(inbox['address'], interval=3, timeout=60)
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

    # Poll until an email arrives (max 60s)
    for i in $(seq 1 20); do
      RESULT=$(curl -sf "https://vanishinbox.com/api/v1/inbox/$ADDRESS" \
        -H "Authorization: Bearer $VANISH_API_KEY")
      COUNT=$(echo $RESULT | jq '.emails | length')
      if [ "$COUNT" -gt "0" ]; then
        echo "Email received"
        echo $RESULT | jq '.emails[0].subject'
        exit 0
      fi
      sleep 3
    done
    echo "Timed out waiting for email"
    exit 1
```

---

## Tips

**One address per test.** Generate a fresh address for each test case. This prevents emails from different test runs appearing in the same inbox.

**Poll at 3–5 second intervals.** Emails typically arrive within 5 seconds of being sent. Polling faster than every 3 seconds burns credits without meaningful benefit.

**Check `expires_in_seconds`.** If this is `null`, no email has arrived yet. If it's below `60`, the inbox is about to expire — generate a fresh address if you need more time.

**Use `DELETE` between tests.** If you reuse an address across test runs, call `DELETE /v1/inbox/{address}` first to clear any leftover mail from the previous run.

**Domain blocking.** If a site rejects a specific domain, fetch `/v1/domains` and try a different one. All five domains accept inbound mail identically.

---

## Links

- [VanishInbox](https://vanishinbox.com) — free web inbox, no API key needed
- [Dashboard](https://vanishinbox.com/dashboard) — manage keys and credits
- [Developer docs](https://vanishinbox.com/developers) — full documentation
- [Contact](https://vanishinbox.com/contact) — support and questions
