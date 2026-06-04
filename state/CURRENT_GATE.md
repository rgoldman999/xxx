# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Fix btg-stt /stream 404 by mounting the STT ws route under a nested prefix (mirrors the backend's WORKING /api/call/ws/{id}). Diff DRAFTED, not applied. Awaiting Rob approval to apply + confirm-before-live redeploy.

## Verified root differentiator (RESULTS 02:03Z)
- Backend /api/call/ws/{id} (prefix /api/call, @router.websocket) WS upgrade REACHES container (Runs status 1006 WS-close). WS works on our v4 apps.
- btg-stt bare /stream -> "/stream GET 404", upgrade not reaching container.
- Differentiator: nested multi-segment path works; bare single-segment fails. (Caveat: scale-to-zero/GPU vs always-on/CPU also differ — fallback suspect if fix fails.)

## DRAFTED diff (NOT applied) — 3 edit sites
1. bridge/main_stt.py:77 (canonical):  app.include_router(stt_svc.router)  ->  app.include_router(stt_svc.router, prefix="/api/stt")
2. ops/cerebrium/stt/main_stt.py:77 (DEPLOYED snapshot — currently identical to canonical): SAME edit as #1. Required or deploy won't pick up the fix.
3. backend/app/services/clients/stt_bridge.py:56-68 (_ws_url_from): build "/api/stt/stream" instead of "/stream" (all three return branches).
- Net route: /stream -> /api/stt/stream (WS). Cerebrium healthcheck unaffected: real healthcheck is bare @app.get("/healthz") at main_stt.py:62 (stays); the router's secondary /healthz harmlessly moves to /api/stt/healthz.
- Local-dev parity preserved (_base_url localhost fallback + bridge both move together). No STT logic change.

## Next step — ROB DECISION
- "apply it" -> executor applies the 3 edits, shows post-edit verification (greps), commits (backtogether repo), then asks for confirm-before-live redeploy of btg-stt + backend.
- Apply is AUTO-RUN-draftable but APPLYING + COMMIT to the product repo + REDEPLOY are gated: executor will apply edits + commit only on "apply it", and will still pause for explicit confirm-before-live before each redeploy.

## Verify after deploy (read-only)
- WS probe to /api/stt/stream -> dashboard Runs shows WS-close status (not GET 404).
- Then real reprocess of a fresh video -> audio_extracted -> transcribed, source_speaker_segments>0.
- If still 404: path-depth disproven -> test btg-stt min_replicas=1 (SHORT ROB-ONLY spend test, then revert) or escalate.

## ROB-ONLY (carried)
- Approve apply + commit to product repo; redeploy (confirm-before-live); min_replicas/spend; upload/reprocess; TTS/face. No secret values read/set by executor.

## Hard constraints
No edits applied / no product-repo commit without "apply it". No deploy without confirm-before-live. No retry of 294c44ea until ws path non-404. No upload/reprocess by executor. No TTS/face.

## Follow-on
- Apply the SAME prefix pattern to the TTS bridge before its bring-up (preempt the identical WS 404 on paid GPU).
- Small fix (draft later): make the STT bridge call attach a message so the failure error string is never blank.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).