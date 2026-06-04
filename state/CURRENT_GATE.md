# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Fix the /stream 404. Docs research done: Cerebrium v4 DOES support WebSockets (custom runtime); our URL shape, uvicorn bind, entrypoint, and healthcheck are all valid per docs. The 404 is platform/deploy-layer — the WS upgrade reaches the gateway as a plain GET. STT still not validated end-to-end.

## Docs research — DONE (RESULTS 01:44Z; sources in RESULTS)
- WS supported on v4 via [cerebrium.runtime.custom]. URL: wss://api.aws.<region>.cerebrium.ai/v4/<project>/<app>/<ws-route>. Ours wss://.../btg-stt/stream is CORRECT.
- Eliminated (docs + source): WS-unsupported (no), wrong URL (no), bad bind/entrypoint (no — uvicorn 0.0.0.0:8003, /healthz 200), readycheck missing (no — TCP fallback is valid), entrypoint list form (no — valid).
- Remaining cause (NOT guessed as fact): deployed revision may not have applied the custom runtime, OR an undocumented Cerebrium WS proxying nuance. Needs a clean redeploy test and/or Cerebrium support.

## Proposed next step (NEEDS ROB APPROVAL — deploy involved)
1. (confirm-before-live) Redeploy btg-stt UNCHANGED to ensure the current custom-runtime toml is the ACTIVE revision. Then re-test /stream via dashboard Runs (read-only).
   - target: app btg-stt (p-a907d7c5), env prod. scope: redeploy only, no toml/code change.
   - command: cd ops/cerebrium/stt && cerebrium deploy --config-file ./cerebrium.toml  (logged bg, as before)
   - smoke: /healthz 200 + dashboard Runs shows a fresh /stream attempt result.
   - rollback/stop: prior revision remains; if /stream still 404, STOP and escalate to Cerebrium support (do not loop redeploys).
2. If still 404 after a clean redeploy: ROB escalates to Cerebrium (support/Discord) with Runs evidence (/healthz 200 vs /stream GET 404), OR we revisit transport. Architectural fallback (DECISION, Rob/Jeannine): move STT off streaming WS to a non-WS path if Cerebrium WS proves unworkable.
3. Optional low-confidence toml alignment (only if Rob wants to try before support): entrypoint -> ["uvicorn","main_stt:app","--host","0.0.0.0","--port","8003"] + readycheck_endpoint="/healthz". Small toml change = AUTO-RUN draft + show-diff; deploy = confirm-before-live.

## Why this is a checkpoint, not auto-run
The only remaining executor action is a redeploy (confirm-before-live by policy). Everything diagnosable read-only is done.

## ROB-ONLY (carried)
- Approve redeploy; Cerebrium support escalation; architectural transport decision; secret/url value changes; upload/reprocess; TTS/face. No secret values read/set by executor.

## Hard constraints
No deploy without explicit Rob approval (confirm-before-live). No code/toml change without show-diff + approval. No retry of 294c44ea until /stream returns non-404. No upload/reprocess/authed-call by executor. No TTS/face.

## Note
- Independent small fix worth doing later: make transcribe_file/open_stream attach a message so the failure's error string is never blank.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
