# Architecture

## Why four layers

The brief's business questions map almost one-to-one onto four distinct concerns, and conflating them is the most common way these systems go wrong (e.g. "fixing" a fact by editing history, or losing evidence when a contradiction is resolved). So the model keeps them as four separate tables with one-directional dependencies:

```
raw_events  --(extraction)-->  facts  --(read-time join)-->  context
     |                            |
     +-> idempotency_index        +-> ambiguous_identities
     +-> ingestion_conflicts
     +-> duplicate_candidates
                                  context_snapshots (for diff)
```

`raw_events` never changes once written. `facts` are derived and can be *recomputed* (the whole table is rebuilt from `raw_events` on each `build_facts()` call) but a fact row is never silently deleted — losing status is itself recorded as `status='superseded'`/`'contradicted'`, not a deletion. `context` is a pure read view computed on demand; nothing in it is persisted as ground truth except the snapshot used for diffing.

## Data model

**raw_events** — one row per ingested event, exactly as received, plus bookkeeping: `body_hash` (sha256 of everything except `event_id` / `idempotency_key`, used to tell a true retry from a conflicting reuse of the same key) and `ingest_status` (`new` / `duplicate_noop` / `idempotency_conflict` / `skipped_already_ingested`).

**idempotency_index** — one row per `idempotency_key` ever seen, mapping it to the first event_id and body hash that claimed it. This is the single source of truth for "have I seen this key before, and with what content."

**ingestion_conflicts** — populated only when a key is reused with a different body. Both the original and the conflicting event remain in `raw_events`; this table just records that the reuse happened and why it matters (a true idempotency violation upstream, worth a paging alert in production).

**duplicate_candidates** — a softer, content-level signal: an event whose payload says `possible_duplicate_of` or `retry_of` another event_id. This is distinct from the idempotency mechanism on purpose — duplicate *meaning* (two different messages about the same incident) and duplicate *delivery* (the same message delivered twice) are different failure modes with different fixes.

**facts** — one row per (entity_type, entity_id, attribute, *source event*). I chose "one fact row per contributing event" rather than "one row per attribute, mutated in place" specifically so that every value ever asserted stays visible and explainable, instead of being overwritten. Columns: `value` (JSON), `status` (`current` / `superseded` / `contradicted` / `corroborating` / `unverified`), `confidence` (0–1), `valid_from` / `valid_to` (temporal windows), `source_event_ids`, `superseded_by`, `reasoning` (human-readable, generated once at resolution time so `explain` never has to recompute it), and `scope_account_id` (non-null only for policy facts — the explicit anti-leakage column).

**ambiguous_identities** — pairs of entity_ids the system suspects might be the same real-world person/account, with a `basis` (`shared_phone` for a shared structured attribute, `text_mention` for free-text mentions) and a permanent `status='needs_review'`. There is no merge action anywhere in the schema — by construction, the system cannot silently merge two entities.

**context_snapshots** — one row per (entity_id, built_at), storing a canonicalized, timestamp-free fact list and its hash. Used purely for the diff/digest feature; not part of any other read path.

## Reliability decisions, explicitly

**Duplicate retries vs. idempotency conflicts.** The two are detected differently and treated differently:
- Same `idempotency_key`, identical body hash → `duplicate_noop`. The raw row is still inserted (we never lose the evidence that a retry was attempted), but it contributes zero new facts, and confidence is not re-derived or bumped from it.
- Same `idempotency_key`, *different* body hash → `idempotency_conflict`. Both versions are preserved and fact-derived as if they were independent events (so we don't lose real information), but the conflict itself is logged loudly and surfaced in `context` under `system_flags`.

**The deliberate trap in the seed data, and how the system handles it.** evt-1008's *text* says "Retry of evt-1004 with the same idempotency key," but its actual `idempotency_key` field (`idem-1008`) is *not* the same as evt-1004's (`idem-1004`) — only evt-1009 actually reuses `idem-1008`. The system does not trust the free-text claim; it keys all idempotency logic off the structured `idempotency_key` field. This produces the (correct, if initially surprising) result that evt-1008 is treated as a brand-new event relative to evt-1004 (flagged only as a softer `duplicate_candidate` via its own `retry_of` payload field), while the *actual* conflict the system raises an alarm on is evt-1008 vs evt-1009. Trusting unstructured text over a structured idempotency key is exactly the kind of "silent guess" the brief says not to make.

**Contradiction scoring is a simple, fully transparent formula** (see `src/resolve.py` docstring for the exact numbers): reliability weight + recency rank, with explicit author signals (`correction`, `supersedes`, `old_fact`, `unverified_policy_guess`) as overrides. A tie between disagreeing values is marked `contradicted`, not resolved by a coin flip. I picked a transparent arithmetic rule over a learned/black-box score because every output is expected to be explainable in the `explain` view.

**Policy scoping / anti-leakage.** Policy facts carry an explicit `scope_account_id` column, set from the policy event's own `scope_account_id` payload field. `context.build_context()` filters policy facts with `WHERE scope_account_id = :queried_account_id` — there is no path by which Nova's policy can appear in Delta's or Helios's context. Delta's own *unverified guess* about having "the same restriction as Nova" is surfaced separately, under `unverified_account_claims`, explicitly labeled as low-confidence and excluded from `policies`.

**Ambiguous identity.** Two independent, low-precision heuristics (shared phone number in payload; text mentions of another known contact's name next to ambiguity language like "same phone" or "do not merge"). Both produce `needs_review` pairs, never a merge. The seed data's chain — Mona Salem ↔ "M. Salem" ↔ Omar Adel, all connected by one shared phone number across two different accounts — is exactly the situation a real support team would want flagged, not silently resolved either way.

## Context build ("what should the rep know before calling Helios?")

`build_context(account_id)` assembles:
- the account's own current/superseded/contradicted facts (superseded ones kept visible but clearly tagged, not hidden)
- every related contact's and ticket's facts, found by scanning `related_entity_ids` in `raw_events`
- policy facts scoped to this account only
- unverified claims about this account, separated from confirmed facts
- ambiguous identity pairs touching any related contact
- system integrity flags (idempotency conflicts, duplicate candidates) touching this account's events

## Diff / digest since last context build (mid-test scope addition)

Implemented, not just specified. `build_context_with_diff()`:
1. builds context as above
2. reduces it to a canonical, timestamp-free list of `(scope, attribute, status, value)` tuples
3. loads the most recent prior snapshot for that `entity_id` from `context_snapshots`, diffs the two tuple lists into `added` / `changed` / `removed` / `unchanged_count`
4. stores the new snapshot for the next call

This is a full implementation, not a stub, because the data and the CLI already supported everything needed. If this needed to scale past "one account's facts fit in memory," the production version would diff against a per-attribute `last_notified_value` column instead of a full snapshot blob (see `NEXT.md`).

## What I intentionally did not build

- **LLM-based extraction.** Rule-based extraction over a fixed payload schema is more accurate and 100% traceable for this seed set. Real production events will have free text that doesn't map cleanly to a payload key, and that needs an LLM extraction step with its own confidence/verification path — out of scope for "smallest credible version."
- **Time-based staleness decay.** Only explicit self-flagged staleness (`old_fact: true`) is implemented; a real per-attribute TTL is specified in `NEXT.md` but not built, since 9 days of seed data can't exercise it meaningfully.
- **Authentication, multi-tenant isolation at the infra level, a real API server.** This is a CLI over SQLite, per the brief's "smallest credible version" instruction. The data model would map directly onto a REST API (each CLI command is already a pure function over a connection) — see `NEXT.md`.
- **Automatic identity merging.** By design. Ambiguity is surfaced, never resolved automatically, per the brief's explicit instruction not to silently guess.
