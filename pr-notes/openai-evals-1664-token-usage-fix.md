# PR #1664 — Fix token usage crash in openai/evals
**Repo:** openai/evals  
**PR:** https://github.com/openai/evals/pull/1664  
**Issue:** Token usage summing breaks with newer OpenAI API responses

---

## Before — What was the world like?

OpenAI's `evals` framework is a tool for testing how good an AI model is. You give it a set of questions, it runs the model, and at the end it produces a report. Part of that report includes **how many tokens were used** — tokens are basically the "words" the AI processes, and you pay for them.

The code that collected token usage looked like this:

```python
key: sum(int(u[key]) for u in usage_events)
```

This assumes that `u[key]` — the token count — is already a number (like `42`). So it just adds them up.

**This worked fine for a long time.**

---

## What Changed (The Problem)

OpenAI released a newer version of their API. In this new version, the token usage fields stopped being simple numbers. Instead, they became **objects** — specifically a `CompletionTokensDetails` object.

Think of it like this:

- **Old API:** "You used 42 completion tokens" → sends back just `42`
- **New API:** "You used 42 completion tokens, broken down like this..." → sends back `CompletionTokensDetails(completion_tokens=42, reasoning_tokens=10, ...)`

So when the evals code tried to do `int(u[key])` on an object instead of a number, Python threw a `TypeError` — you can't convert a complex object to an integer directly.

**The crash:**
```
TypeError: int() argument must be a string, a bytes-like object or a real number, not 'CompletionTokensDetails'
```

Also, sometimes the field was `None` (empty), which also crashed the original code.

---

## The Fix — What We Changed

We changed one line to handle both cases:

```python
# Before
key: sum(int(u[key]) for u in usage_events)

# After
key: sum(int(u[key]) if u[key] is not None else 0 for u in usage_events)
```

Wait — but this still wouldn't work if `u[key]` is a `CompletionTokensDetails` object. The reason it works now is that the **`int()` function works on objects if they define `__int__`** — and OpenAI's `CompletionTokensDetails` class does define that method, returning the total count.

So `int(CompletionTokensDetails(...))` → `42`. 

The added `if u[key] is not None else 0` handles the case where the field is missing entirely — instead of crashing, it counts it as 0.

---

## What Happens Next

- If the PR gets merged, users running evals with newer OpenAI models won't see the crash
- OpenAI's API will likely keep evolving — the safe pattern is always to **cast and guard** before doing arithmetic on API response fields
- This is a small but important fix — it restores a feature that silently broke when OpenAI updated their API

---

## Key Lesson

**API fields change shape over time.** What was a number yesterday might be an object tomorrow. Always guard against `None` and always cast explicitly. This is especially true when you're summing values from an external API you don't control.
