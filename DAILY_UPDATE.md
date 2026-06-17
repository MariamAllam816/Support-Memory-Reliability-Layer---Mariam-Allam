# DAILY_UPDATE.md

**Branch:** `master`
**Commit range:** Full session — all files created in this single work sample
**Date:** 2026-06-18

## What shipped

A working local "Support Memory Reliability Layer" CLI service, in pure Python 3 stdlib (SQLite + argparse + hashlib + json, no external dependencies), implementing all core requirements from the brief:

- **Evidence preservation.** All 20 seed events are ingested into an immutable `raw_events` table, including the deliberately malformed retry (`evt-1008` / `evt-1009`, same `idempotency_key`, different ticket and body) — preserved and flagged as a conflict, not dropped or silently merged.
- **Fact derivation.** `extract.py` + `resolve.py` derive scoped, versioned facts from event payloads and signal keywords (`correction`, `supersedes`, `old_fact`, `retry_of`), with every fact's status set to `current / superseded / contradicted / corroborating / unverified`.
- **Traceability.** Every fact links back to its supporting `event_id`s; `explain_fact()` shows a fact plus its full evidence and any competing facts with their own reasoning.
- **Scoping.** Facts are scoped by `entity_type` + `entity_id`. Policy facts are additionally scoped by `scope_account_id`, verified to never leak — Delta Stores' context never shows Nova's `no_training` / `no_cross_account_analytics` policy, only its own low-confidence unverified guess.
- **Idempotency / duplicate handling.** `ingest_events()` checks `idempotency_key` + a body hash (excluding `event_id`/`idempotency_key` themselves): true retries are no-ops, replays of the same `event_id` are skipped, and same-key-different-body is flagged as an `ingestion_conflicts` row rather than silently overwritten.
- **Ambiguous identity detection.** Two heuristics (shared-phone-in-payload, and text patterns like "same phone" / "do not merge") flag the Mona Salem / M. Salem / Omar Adel triangle as `needs_review` — never auto-merged.
- **Compact context + diff/digest.** `build_context()` answers "what should a rep know before calling Helios" with current facts, contradicted/stale facts called out, ambiguous identities, and system flags. `build_context_with_diff()` (added for the mid-test scope change Lina raised) hashes and diffs against the last stored snapshot, listing exactly which attributes changed since the previous context build.

## Tests / results

19 unittest tests, pure stdlib (`python3 -m unittest tests.test_memory -v`): **19/19 passing.** Coverage includes:
- Idempotency conflict detection (evt-1008 vs evt-1009)
- True-retry no-op (same key, same body)
- Event-id replay skip
- Contradiction resolution (Helios plan Starter→Enterprise Support)
- Stale self-flagged fact handling (evt-1018)
- Explicit correction winning (affected seats 42→48)
- Explicit supersede (Mona's contact preference WhatsApp→email)
- Ambiguous identity flagging (Mona/M. Salem/Omar shared phone)
- Unverified-policy-guess handling (Delta's low-confidence guess)
- Account context scoping
- Cross-account policy non-leakage
- Explain-view competing-facts display
- Diff/digest behavior across first build / no-op rebuild / real change

Four real bugs were found and fixed during the session (see `AI_USAGE.md` for detail):
1. The deliberate trap — trusting text over structured fields
2. Unverified policy guess not being marked as unverified
3. Double-JSON-parsing in explain view test
4. Duplicate candidate query missing related entities

## Files changed

```
data/events.json                    # Real seed data (478 IDs)
src/__init__.py, cli.py, context.py, extract.py, identity.py,
    ingest.py, resolve.py, store.py  # Core implementation
tests/__init__.py, test_memory.py   # 19-test verification suite
sample_outputs/*.txt, *.json        # 9 real CLI outputs
README.md, ARCHITECTURE.md,
    AI_USAGE.md, DAILY_UPDATE.md, NEXT.md  # Documentation
```

## Blockers

None that stopped delivery. All four issues above were caught and fixed within the session, not left open.

## Scope change handled
Lina clarified: true retries (same idempotency key + same body) must return the same logical result WITHOUT creating a second raw event or re-confirming facts. Implemented by adding `continue` in `ingest.py` when `stored_hash == body_hash` — the retry is counted in stats but skipped entirely (no raw insert, no duplicate candidates, no fact re-derivation). Test `test_true_retry_is_noop` updated to verify raw event count stays constant.

## Next step

See `NEXT.md` for the prioritized production roadmap. The single highest-value next step if continuing tomorrow: incremental fact rebuild (today's `build_facts()` does a full deterministic replay of all raw events on every ingest, which is the right call for a 20-event seed set but won't scale — see `NEXT.md` item 1 for the scoping approach).
