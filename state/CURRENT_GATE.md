# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Fix /stream 404 (WS upgrade not reaching container on Cerebrium v4). Unchanged redeploy did not fix it. A web claim about a fixed "/ws" path is disproven, but it surfaced two real code/toml leads. DECISION: escalate to Cerebrium support, or draft+test the highest-probability code fix. STT blocked end-to-end.

## Done / verified
- Unchanged redeploy (build-0844d7dd): /healthz 200, /stream still 404 (dashboard Runs 18:52:02). Rules out stale-revision. (RESULTS 01:53Z)
- Web "/ws fixed ingress path" claim DISPROVEN by Cerebrium source: ws route name is arbitrary (URL segment = your websocket route). Twilio example uses /ws only as its chosen name. (RESULTS 01:59Z)

## Two real leads (source-grounded, UNCONFIRMED — both code/toml, gated)
1. WS mounting: working examples mount the ws route directly on app; ours uses an APIRouter websocket route + app.include_router (stt.py:205, main_stt.py:77). Whether Cerebrium ingress WS negotiation is sensitive to router-vs-app mount is unverified. HIGHEST-probability fix candidate.
2. Entrypoint form: examples use entrypoint=["uvicorn","main_stt:app","--host","0.0.0.0","--port",8003]; ours is ["python3","-u","main_stt.py"]. Both bind 0.0.0.0:8003 (/healthz 200). Deviation from documented pattern.

## DECISION NEEDED (Rob) — pick one
A. ESCALATE to Cerebrium support (ROB-ONLY): send evidence pack (RESULTS 01:53Z) + the two leads. Lower-risk, definitive. STT stays blocked until they answer.
B. TRY highest-probability fix first: executor DRAFTS lead #1 (mount ws route directly on app, and/or align entrypoint) as show-diff for Rob approval; then confirm-before-live redeploy; re-test /stream via dashboard Runs. Faster if hypothesis holds; if still 404, fall back to A.
   - If B: change is small + reversible (route decorator / include_router; optionally entrypoint). NO behavior change to STT logic. Draft only — nothing applied without approval.

## Productive work available regardless
- Small AUTO-RUN code fix (draft+show-diff): make the STT bridge call attach a message so the failure error string is never blank.
- Do NOT start TTS until WS routing is solved — TTS likely streams over WS and would hit the same wall on paid GPU.

## ROB-ONLY (carried)
- Cerebrium escalation; approve any code/toml change (show-diff first); redeploy (confirm-before-live); transport/arch decision; secret/url changes; upload/reprocess; TTS/face. No secret values read/set by executor.

## Hard constraints
No code/toml change without show-diff + approval. No route rename on the disproven "/ws" claim. No deploy without confirm-before-live. No retry of 294c44ea until /stream non-404. No upload/reprocess by executor. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).