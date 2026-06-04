# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
The /api/stt prefix fix is FALSIFIED — /api/stt/stream still logs GET 404. Path-depth was not the cause. Next: test the scale-to-zero hypothesis (or escalate). STT blocked end-to-end.

## Done / verified (RESULTS 02:14Z)
- btg-stt redeployed with prefix fix (build-da6a8e30, live). /healthz 200.
- /api/stt/stream -> dashboard Runs "GET 404" (19:13:18), same failure mode as bare /stream. PATH-DEPTH FALSIFIED.
- Backend NOT deployed (HALT honored). No reprocess.
- Code: prefix fix committed (backtogether d30301e) but ineffective. Recommend LEAVE (not wrong, just insufficient) unless Rob wants revert.

## Leading suspect now: scale-to-zero vs always-on
- Working WS = backend, CPU, min_replicas=1 (always-on). Broken WS = btg-stt, GPU, min_replicas=0 (scale-to-zero).
- The 404 hit an already-warm container (same as prior /healthz 200), failed fast (26ms) — not a naive cold-start drop. Possibly scale-to-zero apps get a sync-request routing profile that does not proxy WS upgrades. UNVERIFIED.

## DECISION (Rob) — pick
A. Test scale-to-zero: set btg-stt min_replicas=1 + redeploy, re-probe /api/stt/stream via Runs. Cleanest test of the suspect. COST: always-on ADA_L4 ~$0.80/hr (~$576/mo) — exceeds $100 ceiling, so SHORT test then revert to 0. ROB-ONLY (spend approval).
B. Escalate to Cerebrium support: evidence now stronger (WS works on always-on CPU app, 404s on scale-to-zero GPU app, same project + same @router pattern). Definitive, slow.
C. Both: A as fast test, B in parallel.

## If A and it WORKS (WS reaches container on min_replicas=1)
- Confirms scale-to-zero is the cause. Then the real tradeoff is cost: always-on ADA_L4 blows the $100 ceiling. Options to decide then: keep min_replicas=1 and accept spend; or find a Cerebrium scale-to-zero+WS config; or move STT transport off WS. DECISION for Rob/Jeannine.

## ROB-ONLY (carried)
- min_replicas/spend change; Cerebrium escalation; backend redeploy; upload/reprocess; revert decision; TTS/face. No secret values read/set by executor.

## Hard constraints
No min_replicas/spend change without explicit Rob approval. No backend deploy / reprocess until /api/stt/stream (or whatever path) returns non-404. No push unless asked. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).