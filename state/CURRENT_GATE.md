# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Fix btg-stt /stream 404. Root differentiator now VERIFIED: WS works on our v4 apps when the route is a nested multi-segment path (backend /api/call/ws/{id} -> reaches container), and fails when it's a bare single-segment path (btg-stt /stream -> 404). Proposed fix: give the STT ws route a prefix. Needs Rob approval (code change + redeploy).

## Verified (read-only, RESULTS 02:03Z)
- Backend's own ws route /api/call/ws/{call_id} (mounted prefix="/api/call", @router.websocket): WS upgrade REACHES container — dashboard Runs logged status 1006 (WS close), not 404. => WebSockets WORK on our v4 apps.
- btg-stt /stream (no prefix, bare): dashboard logs "/stream GET 404" — upgrade does NOT reach container.
- ELIMINATED: router-vs-app mounting (backend uses @router+include_router too and WORKS); "/ws magic path" (disproven).
- DIFFERENTIATOR: nested multi-segment path (works) vs bare single-segment path (fails).
- CAVEAT (not proven causal): apps also differ in scale-to-zero (btg-stt min_replicas=0, GPU) vs always-on (backend min_replicas=1, CPU). Path-depth is lowest-risk first fix; scale-to-zero is the fallback suspect.

## Proposed fix — NEEDS ROB APPROVAL (draft first, then confirm-before-live)
Plan (option B-style):
1. (AUTO-RUN draft, show-diff, NOT applied) Edit bridge to mount the STT router under a prefix mirroring the backend: include_router(stt_svc.router, prefix="/api/stt") so the ws route becomes /api/stt/stream. Edit backend client _ws_url_from (stt_bridge.py:56-72) to build the new path. Two small reversible edits; no STT logic change.
2. (ROB approves diff) apply.
3. (confirm-before-live) redeploy btg-stt + backend.
4. (verify read-only) probe new ws path -> dashboard Runs shows WS-close status (not GET 404); then a real reprocess of a fresh video -> transcribed, segments>0.
5. If still 404: path-depth disproven -> test scale-to-zero hypothesis (btg-stt min_replicas=1 as a SHORT ROB-ONLY spend test, then revert) or escalate.

## DECISION (Rob) — pick one
- "draft it" -> executor produces the show-diff for the prefix fix (no apply).
- or adjust the prefix choice (e.g. /v1 instead of /api/stt).

## ROB-ONLY (carried)
- Approve code change (show-diff first); redeploy (confirm-before-live); min_replicas/spend change; upload/reprocess; TTS/face. No secret values read/set by executor.

## Hard constraints
No code/toml change applied without show-diff + approval. No deploy without confirm-before-live. No retry of 294c44ea until /stream-equivalent non-404. No upload/reprocess by executor. No TTS/face.

## Productive work available regardless
- Small fix (draft+show-diff): make the STT bridge call attach a message so the failure error string is never blank.
- Do NOT start TTS until WS routing is fixed (TTS likely streams over WS — same risk on paid GPU; AND the same prefix fix should be applied to the TTS bridge preemptively).

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).