# PR #6922 — Respect Retry-After header for 429 rate limits in Weave Node SDK
**Repo:** wandb/weave  
**PR:** https://github.com/wandb/weave/pull/6922  
**Issue:** Node SDK retries 429 responses without respecting server's requested wait time

---

## Before — What was the world like?

The **Weave Node SDK** is a TypeScript/JavaScript library that lets Node.js apps talk to the Weave server. When your app calls `weave.log(...)` or similar, the SDK sends an HTTP request to the server.

Sometimes the server is too busy and responds with **HTTP 429 — Too Many Requests**. This means: "slow down, you're sending too many requests."

The SDK already had a retry mechanism — if a request failed, it would wait a bit and try again. The waiting strategy was **exponential backoff**: wait 100ms, then 200ms, then 400ms, 800ms... doubling each time up to a maximum.

This is a good general strategy. But there was a gap.

---

## What was the Problem?

When a server sends back a 429, it can also include a **`Retry-After` header**. This header tells the client exactly how long to wait:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

This means: "wait 30 seconds, then try again."

The old SDK ignored this header completely. It would just use its own exponential backoff — which might be 100ms — and immediately retry. The server just told it to wait 30 seconds, but the SDK retried in 0.1 seconds. This means:
- The retry also gets rejected with 429
- The SDK keeps retrying, getting rejected, retrying...
- It wastes all its retry attempts without ever actually waiting long enough
- The request ultimately fails, even though it would have succeeded if the SDK had just waited as the server asked

---

## The Fix — What We Changed

We added a function `parseRetryAfterMs()` that reads the `Retry-After` header from a 429 response:

```typescript
function parseRetryAfterMs(response: Response): number | null {
  const header = response.headers.get("Retry-After");
  if (!header) return null;

  // Format 1: "Retry-After: 30"  (seconds as a number)
  const seconds = parseFloat(header);
  if (!isNaN(seconds) && seconds >= 0) return seconds * 1000;

  // Format 2: "Retry-After: Thu, 22 May 2026 10:00:00 GMT"  (a specific date/time)
  const date = new Date(header);
  if (!isNaN(date.getTime())) return Math.max(0, date.getTime() - Date.now());

  return null;
}
```

The `Retry-After` header has two possible formats — either a number of seconds (`30`) or a specific date/time. We handle both.

Then in the retry logic, when we get a 429:

```typescript
if (response.status === 429 && retryOnStatus(response.status)) {
  const retryAfterMs = parseRetryAfterMs(response);
  if (retryAfterMs !== null && retryAfterMs > 60_000) {
    // Server wants us to wait more than 60 seconds — give up, don't waste retries
    return response;
  }
  const delay = retryAfterMs ?? Math.min(baseDelay * 2 ** attempt, maxDelay);
  // wait and retry...
}
```

The **60 second threshold** is important: if the server says "wait 2 hours," we don't actually wait 2 hours. We return the 429 immediately so the caller can handle it. But if it says "wait 30 seconds," we respect that.

---

## The Codex Review — Two Issues Found

After the PR was opened, an automated code reviewer (Codex) flagged two valid problems:

**Issue 1:** The 429 logic was bypassing `retryOnStatus`
- The function accepts a callback `retryOnStatus` that lets callers decide which status codes to retry
- The old fix checked for 429 *before* asking `retryOnStatus` — meaning the Retry-After logic ran even if the caller said "don't retry 429s"
- Fix: wrap the 429 logic in `if (retryOnStatus(response.status))`

**Issue 2:** Negative `Retry-After` values were accepted
- If the header said `-5`, `parseFloat("-5")` returns `-5`, and `-5 >= 0` is false — so we'd fall through to the date parsing, fail, and return `null`
- Actually this was fine, but we added an explicit `seconds >= 0` guard to make the intent clear

Both issues were fixed in a follow-up commit with a comment explaining each change.

---

## What Happens Next

- If merged, the Node SDK correctly respects the server's rate limit instructions
- The SDK won't waste its retry budget hitting a server that clearly isn't ready yet
- This pattern — read `Retry-After`, use it if reasonable, give up if it's too long — is the standard way to handle 429s in any HTTP client

---

## Key Lesson

**HTTP headers are instructions from the server.** When the server sends `Retry-After`, it's not a suggestion — it's telling you exactly what you need to do to succeed. Ignoring it and using your own timing is like someone saying "call me back in an hour" and you calling back every 5 seconds. Follow the protocol.
