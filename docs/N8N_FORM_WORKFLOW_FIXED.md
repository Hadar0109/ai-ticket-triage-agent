# Fixed Form-Trigger Workflow (Batch 4 — separate nodes)

**File:** `n8n/ticket_triage_form_fixed.json`

## Canvas layout (all process nodes visible)

```
Form → Read CSV → Edge Case → Limit
  → Loop Over Items (batch 4)
      loop → Batch Pause (2s)
           → Route: Skip or Process
                ├─ process → Gemini LLM Call → Parse → Fraud Validation ─┐
                ├─ skip    → Format Skipped Ticket ─────────────────────┤
                └─ error   → Retry Counter → Retry or Give Up? ─────────┤
                              └─ backoff → Gemini (retry)               │
                                                                          ▼
                                                              Batch Loop Back
                                                                          │
                                                                          ▼
                                                                    Loop (once/batch)
      done → Format CSV → Convert to Binary File
```

## Why batch 4 without duplication

`batchSize: 4` is fast **only if the loop receives one continuation per batch**, not one per ticket.

**Fix:** **Batch Loop Back** (`Run Once for All Items`) sits **between** processing and the loop. Every path (success, skip, give-up) feeds into it; it returns the whole batch in **one** execution.

Do **not** wire Parse / Fraud / Clean Up directly to Loop Over Items.

## Speed vs rate limits (15 tickets)

| Setting | Approx. time |
|--------|----------------|
| Batch size 4, 2.5s between API calls | 4 batches × ~7.5s ≈ **30s** of API spacing (+ API latency) |
| Old batch size 1, 8s wait | ~**120s** minimum |

Tune `API_DELAY_MS` in **Process Batch (4)** (default `2500`). If you still see 429s, use `3000`–`3500`.

## Configuration

| Node | Setting |
|------|---------|
| **Loop Over Items** | Batch Size = **4** |
| **Gemini LLM Call** | Batching: size **1**, interval **2500** ms |
| **Batch Pause (2s)** | One wait per batch before routing |
| **Limit** | 15 for testing; remove for full CSV |

## Import

1. Import `ticket_triage_form_fixed.json`.
2. Attach **x-goog-api-key** on **Gemini LLM Call**.
3. Disable the old workflow to avoid duplicate form triggers.

## Expected result

- 15 tickets → **15 rows** in one CSV (not 64).
- **4 loop iterations** for 15 tickets (4+4+4+3).
- Skipped empty tickets never call Gemini.

## Nodes checklist

| Node | Role |
|------|------|
| Gemini LLM Call | HTTP POST to Gemini (credential here) |
| Parse + Validate Response | JSON parse + fallbacks |
| Fraud Validation Layer | Keyword overrides |
| Retry Counter + Backoff Calc | 429 / API errors |
| Route: Retry or Give Up? | Max 3 retries |
| Exponential Backoff Sleep | 2s, 4s, 8s… |
| Format Skipped Ticket | Empty tickets (no API) |
| **Batch Loop Back** | **Required** — one loop-back per batch |
