# PR #1669 — Hide stack traces from API responses in openai/evals
**Repo:** openai/evals  
**PR:** https://github.com/openai/evals/pull/1669  
**Issue:** Flask app returns full Python error details to API clients

---

## Before — What was the world like?

Inside openai/evals, there's a mini web server built with **Flask** (a Python web framework). This server runs inside a Docker container and acts as a browser-controlling tool — it lets the AI agent click buttons, fill forms, and navigate websites as part of multi-step web tasks.

When something goes wrong in this server, Python raises an **exception** — an error object that contains:
- The type of error (e.g. `ValueError`)
- The error message (e.g. `"URL not found"`)
- The **full stack trace** — a list of every function call that led to the crash, with file paths and line numbers

The original code was doing this in error responses:

```python
return jsonify({"error": f"Failed to navigate: {e}"}), 500
```

That `{e}` — which looks harmless — actually dumps the **entire exception** as a string into the API response.

---

## What was the Problem?

This is a **security vulnerability**. When you send internal error details to an API client, you're handing them a map of your system:

- **File paths** reveal your server's directory structure (e.g. `/app/routes/browser.py`)
- **Line numbers** pinpoint exactly where to look for weaknesses
- **Error messages** can reveal database names, internal URLs, config values
- **Library names and versions** tell attackers which known CVEs to try

This is called **information disclosure** — you're accidentally telling the outside world more than they need to know.

In security terms: the client only needs to know *"something went wrong."* The developer needs to know *why*. Those are two different audiences and should get two different messages.

---

## The Fix — What We Changed

We changed 4 locations in the Flask app. The pattern was the same each time:

```python
# Before — leaks internal details
return jsonify({"error": f"Failed to navigate: {e}"}), 500

# After — generic message to client, full error logged internally
logger.error(f"Navigation failed: {e}")
return jsonify({"error": "Navigation failed"}), 500
```

Two things happen now:
1. The **client** gets a clean, generic message — enough to know it failed, nothing more
2. The **server logs** get the full error — so developers can still debug it

---

## What Happens Next

- If merged, the evals server no longer leaks internal paths or error details
- The logs are still full — nothing is lost for debugging
- This pattern (log internally, return generic externally) is the **standard security practice** for any web API

---

## Key Lesson

**Never put `{e}` or `{exception}` directly into an API response.** It's one of the most common accidental security mistakes in web development. The rule is simple:

- `logger.error(f"Something failed: {e}")` → for you (the developer)
- `return jsonify({"error": "Something failed"})` → for the client

Keep those two things separate. Always.
