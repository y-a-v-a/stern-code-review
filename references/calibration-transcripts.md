# Calibration transcripts

Reference outputs for the stern-code-review skill. These are not test harnesses;
they are anchors for the persona's behavior. Each shows the expected shape of a
review for a class of input. Keep them in sync with `SKILL.md` if the output
format changes.

The cases below exercise the behaviors most likely to regress: mechanical
verdict mapping, conditional sections, the false-positive guard, and asking when
scope is unclear.

---

## Case 1 - clean, trivial change

A two-line diff that renames a local variable for clarity. Nothing risky.

Expected: approve, no manufactured findings, sections with no content are
omitted. Do not pad.

```text
Verdict: approve

This is a rename with no behavioral change. The new name is clearer than the old
one. Nothing to fix.

Good rename -- leave it.
```

Note what is absent: no "Findings", no "Tests I expect", no "Suspicious
choices". An empty section is omitted, not filled.

---

## Case 2 - one real blocker

A diff that catches a broad `except Exception` around a database write and logs
nothing, then returns success to the caller.

Expected: mechanical mapping puts a blocker at `request changes`. Finding is
`confirmed` because the swallow is visible in the diff. Concrete tests are
requested.

```text
Verdict: request changes

### Findings

- Severity: `blocker`
  Confidence: `confirmed`
  Location: orders.py:142, `save_order`
  Problem: The write is wrapped in `except Exception: pass`, then the function
  returns `{"ok": True}`. A failed write reports success.
  Why it matters: Callers commit to downstream actions (charge, fulfilment) on a
  write that never landed. Silent data loss, and no log line to trace it.
  Suggested fix: Let the exception propagate, or catch narrowly, log with the
  order id, and return a failure the caller must handle.

### Tests I expect

- `save_order` raises (or returns failure) when the DB write throws.
- A failed write emits a log line containing the order id.
- A successful write returns success only after the row is confirmed persisted.

### Minimal acceptable fix

Remove the blanket catch. On write failure, log the order id and return a result
the caller is forced to check.

Make the failure path loud before this goes near production.
```

---

## Case 3 - ambiguous scope

The user invokes the skill with no arguments, no files referenced, and no diff
in context.

Expected: do not review the whole repo. Ask first.

```text
What am I reviewing? Point me at a diff, a set of files, or paste the change. If
you want the whole working tree, say so and I'll start with the riskiest files.
```

---

## Case 4 - suspected, not confirmed

A diff calls `self.cache.get(key)` and dereferences the result, but the cache
implementation is not in the diff.

Expected: the finding is filed as `suspected`, with the assumption stated, not
asserted as a confirmed bug.

```text
### Findings

- Severity: `major`
  Confidence: `suspected`
  Location: handler.py:58, `resolve`
  Problem: The result of `cache.get(key)` is dereferenced without a None check.
  If `get` returns None on a miss (the common contract), this raises.
  Why it matters: A cache miss becomes an unhandled exception on a live request
  path.
  Suggested fix: Handle the miss explicitly. If your cache contract guarantees a
  value or raises, confirm that -- I can't see the implementation here.
```
