# NEXT.md — What I'd build next if this became production

This is the smallest credible slice that proves the architecture. If Lina greenlit this for production, here is what I'd prioritize, roughly in order.

## 1. Incremental fact rebuild instead of full rebuild
Today `build_facts()` deterministically replays **every** raw event on every ingest call. That's intentional for this exercise — it's the easiest thing to verify and reason about, and it removes a whole class of bugs around partial updates. It does not scale past a small per-account event log.

What I'd build: a per-entity "rebuild scope" — when new events arrive for `entity_id=ticket_h_478_p1`, only re-run resolution for facts whose `scope` touches that entity (plus anything that reads cross-entity, like policy scope checks). I'd key this off `entity_type` + `entity_id` + `related_entity_ids` so a new ticket event doesn't force a full-account replay. The current `resolve_thread()` function is already scoped per `(entity_type, entity_id, attribute)` group, so this is mostly a matter of computing the affected groups for a new event instead of recomputing all groups.

## 2. Time-based staleness / TTL decay
Right now "staleness" is almost entirely signal-driven: an event explicitly says `old_fact: true`, or a newer event with equal/higher score supersedes an older one. There's no notion of "this fact hasn't been reconfirmed in 90 days, treat it as lower-confidence even though nothing has explicitly contradicted it." For something like account plan or contact preference, that matters — a plan fact from a year-old contract should probably surface as "needs reconfirmation" even if no one ever sent a correction.

What I'd build: a configurable per-attribute TTL (e.g., contract-derived plan facts: 180 days; chat-derived preferences: 60 days) applied at context-build time, not at fact-resolution time — so the underlying fact history stays untouched and "is this stale" is a read-time question, consistent with the rest of the design.

## 3. LLM-based extraction for free text
`extract.py` is rule-based: it pulls structured `payload` keys against a per-entity-type whitelist, and only lightly touches `text`. That works for this seed set because the payloads are clean. Real event exports won't always have clean payloads — sometimes the only signal is in the prose (e.g., an agent note that mentions a plan change in passing with no payload field for it).

What I'd build: an LLM extraction pass that proposes candidate facts from `text` with a required citation back to the literal substring it's drawn from, written into a `status='unverified'` fact (same path the current low-reliability/unverified guesses already use) rather than being trusted automatically. A human or a second deterministic check would need to promote it before it's treated as current. This keeps the "don't silently guess" rule intact even with a fuzzier extractor.

## 4. REST API wrapper
The CLI is sufficient to prove the architecture and is trivially scriptable for sample outputs. For an actual assistant integration, I'd wrap `context.py` and `ingest.py` behind a minimal REST API (`POST /events`, `GET /context/:entity_id`, `GET /facts/:fact_id/explain`, `GET /flags`) using the same SQLite-backed core — no architecture change, just a thin HTTP layer.

## 5. Identity-merge human-approval workflow
`ambiguous_identities` rows are flagged with `status='needs_review'` and never auto-merged — that's correct conservative behavior. But there's currently no workflow to actually resolve them. What I'd build: a small review queue (`GET /flags/identities`, `POST /identities/:id/resolve {decision: merge|reject, reviewer, note}`) that writes a new raw "system" event recording the human decision, so the merge itself goes through the same evidence trail as everything else rather than being a side-channel database edit.

## 6. Per-attribute change tracking for the diff/digest
The current diff/digest (`build_context_with_diff`) hashes the whole canonicalized context and compares it to the last snapshot — good enough to answer "did anything change since last time" and to list which attributes changed, by recomputing per-attribute tuples and diffing the sets. For a real "what changed since I last looked" feed at scale, I'd move to storing each snapshot as individual `(scope, attribute, value, status)` rows with a snapshot version number, so the diff query is a real SQL set-difference instead of an in-memory JSON diff, and so history of changes (not just last-vs-current) is queryable.

## Smaller items
- Structured logging/audit trail for every `ingest` call (who/what/when), separate from the events themselves.
- Configurable reliability weights instead of the hardcoded `high=30/medium=20/low=10` map, since that's a product decision Lina's team should be able to tune.
- A bulk "explain account" view that runs `explain_fact()` over every current fact for an account in one call, instead of one fact at a time.
