# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Build the local handoff COORDINATOR (Phase 1, CLI-only) for the xxx repo. Advise-not-authorize model: the coordinator reads/classifies/records and commits handoff-doc changes; it NEVER authorizes or performs production-mutating actions. (Bridge-tier bring-up — Gate "STT" — is PAUSED behind a ROB-ONLY item; see below.)

## Coordinator-build gate (Phase 1, accepted 2026-06-03)
Build CLI-only; no MCP wrapper, no daemon/watcher, no UI automation.

### Deliverables
- tools/handoff.py — stdlib-only CLI: get-gate, get-next-reply, classify, write-result, update-gate, mark-consumed
- tools/README.md — run + safety model
- state/NEXT_REPLY.md — template if absent
- state/RESULTS.md — template if absent

### Hard properties (enforced in code)
- repo jail: only ~/Projects/xxx, branch main
- path jail: reads/writes only state/*.md
- commit jail: may stage ONLY state/*.md, tools/handoff.py, tools/README.md
- git pull --ff-only before reads/writes
- writes DRY by default; --commit required for write-result / update-gate / mark-consumed
- classify_action is ADVISORY only — returns AUTO-RUN/CHECKPOINT/ROB-ONLY + verdict (PROCEED-ADVISORY|STOP); never authorization
- ROB-ONLY -> STOP
- redact obvious token/key/secret patterns on read; refuse to commit likely secrets
- NO deploy / secrets / DB / calls / app-repo commands — those code paths do not exist in the tool

### Classification of this gate's work
AUTO-RUN — building handoff tooling + docs in the xxx repo, committing tools/handoff.py + tools/README.md + state/*.md. No production surface.

## PAUSED — STT bridge bring-up (ROB-ONLY, awaiting Rob)
Drafted (held, not committed to backtogether): ops/cerebrium/stt/cerebrium.toml (ADA_L4, scale-to-zero min_replicas=0/max_replicas=1, port 8003, /healthz, cuDNN base, ffmpeg). Spend priced: L4 ~$0.80/hr active; scale-to-zero keeps under the $100 ceiling Rob approved. REMAINING ROB-ONLY before deploy: (1) Rob sets BRIDGE_AUTH_SECRET on btg-stt app + backend (value never via chat); (2) Rob confirms config; (3) deploy is confirm-before-live. Resume after coordinator Phase 1.

## Hard constraints (unchanged)
No deploy. No app-repo code changes without show-diff-hold. No env/secret changes. No prod DB writes. No calls placed/ended by executor. Coordinator commits handoff-doc + its own tool files only.

---

## D-2 ARCHIVE — CLOSED (verified with formatter caveat)
Direct API/WS test by USER 2026-06-03. call_id e1f9cdd4-6abc-4469-81b7-4d69637fc6fa; first prod calls row for c447365d… (persona c40776dd, audio); llm_provider_route_decision EMITTED (call.py:582). basicConfig drops extra={} so literal override=qwen not in log text; verified via chain (row user_id=c4473 + resolver returns qwen + 0 non-allowlisted). Active call e1f9cdd4… status=active to be ended (user-run).

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear broad-scope wrangler OAuth session on Mac.
