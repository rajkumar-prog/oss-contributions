# PR #6871 — Fix recursive LiteralOperation schema in wandb/weave OpenAPI spec
**Repo:** wandb/weave  
**PR:** https://github.com/wandb/weave/pull/6871  
**Issue:** #5619 — Recursive `$ref` causes infinite loop in Mintlify docs

---

## Before — What was the world like?

**Weave** is W&B's tool for tracing and evaluating AI model calls. When you call an LLM in your code and want to log what happened, Weave captures it.

Weave has a public API — meaning other programs can talk to it over HTTP. To describe what that API looks like (what endpoints exist, what data they accept), they use an **OpenAPI spec** — a JSON file called `weave.openapi.json`. This file is like a contract: "here's exactly what you can send me and what I'll send back."

One of the types in this spec is called `LiteralOperation` — it represents a literal value in a query (like the number `42` or the string `"hello"`). It's part of a query language Weave uses to filter and search trace data.

---

## What was the Problem?

Inside the OpenAPI spec, `LiteralOperation` was defined like this:

```json
"LiteralOperation": {
  "type": "object",
  "properties": {
    "literal_": {
      "anyOf": [
        { "$ref": "#/components/schemas/LiteralOperation" },  ← points to itself!
        { "type": "array", "items": { "$ref": "#/components/schemas/LiteralOperation" } }  ← also itself!
      ]
    }
  }
}
```

See the problem? `LiteralOperation.literal_` can be... another `LiteralOperation`. Which can be another `LiteralOperation`. Which can be another `LiteralOperation`... **forever**.

This is called a **recursive schema** — the type contains itself with no way to stop.

When **Mintlify** (the tool that generates the Weave documentation website) tried to render this type, it went into an infinite loop trying to expand the schema. It kept saying "what does `LiteralOperation` look like? It has a field that is a `LiteralOperation`. What does that look like? It has a field that is a `LiteralOperation`..." — and crashed with warnings.

---

## The Fix — What We Changed

We broke the recursive loop by replacing the self-references with open-ended types:

```json
# Before (recursive)
"literal_": {
  "anyOf": [
    { "$ref": "#/components/schemas/LiteralOperation" },
    { "type": "array", "items": { "$ref": "#/components/schemas/LiteralOperation" } }
  ]
}

# After (non-recursive)
"literal_": {
  "anyOf": [
    { "additionalProperties": true },      ← "any object" — no specific shape required
    { "type": "array", "items": {} }       ← "any array" — items can be anything
  ]
}
```

- `"additionalProperties": true` means: "this field can be any object with any shape"
- `"items": {}` means: "this array can contain any values"

This is slightly less precise — we're no longer saying exactly what the structure looks like — but it's **accurate enough** for documentation purposes and it doesn't cause infinite loops.

---

## What Happens Next

- Mintlify can now render the Weave API documentation without crashing
- The spec is slightly looser than before (we lost some type precision), but this is a reasonable trade-off for a recursive type that has no natural stopping point
- Future work: Weave's schema generation could be improved to produce a proper bounded recursive schema using `$defs` with depth limits

---

## Key Lesson

**Recursive data structures need a base case** — a point where the recursion stops. In code, that's handled by if-statements at runtime. In static schemas (like OpenAPI), there's no runtime — the schema must be finite. When you have a type that genuinely can nest itself (like a JSON value that can contain any JSON), the right approach is to use open-ended types (`additionalProperties`, `{}`) instead of recursive `$ref`.
