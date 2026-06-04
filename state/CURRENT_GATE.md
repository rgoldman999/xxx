# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
STT /stream returns 404 at Cerebrium's routing layer despite valid WS config. Unchanged redeploy did NOT fix it. Next: Cerebrium support escalation (ROB-ONLY) OR a transport decision. STT blocked end-to-end.

## Done / verified (RESULTS 01:53Z)
- Unchanged redeploy (option 1, Rob-approved) build-0844d7dd LIVE; clean startup; toml unchanged (46d523a).
- Smoke: /healthz 200 (cold start 33.6s). /stream: edge WS handshake returns 101 (server: envoy) BUT dashboard Runs logs the request as "/stream GET 404" (18:52:02, container 6fc5668ff7-kgdk9). Same as pre-redeploy.
- KEY: the 101 is edge-level; Cerebrium routing records/routes /stream as a plain GET -> 404, upgrade NOT reaching the container ws route. A socket 101 is NOT proof of e2e WS; the Runs 404 is authoritative. Redeploy rules out stale-revision/custom-runtime-not-applied.
- => NOT a repo-fixable issue. Escalate to Cerebrium.

## Next step — ROB-ONLY: Cerebrium support escalation
- Rob sends Cerebrium (support/Discord) the escalation evidence pack recorded in RESULTS 01:53Z. Core question: why is a wss:// request to a custom-runtime app's @app.websocket route routed/logged as a plain GET and 404'd to the container on v4? Required flag/config for WS upgrade proxying, or different host/path for WS?
- Until Cerebrium responds, STT end-to-end is blocked. Do NOT loop redeploys. Do NOT retry 294c44ea.

## Parallel options (Rob/Jeannine decision, NOT executor-initiated)
- If Cerebrium WS proves unworkable on v4 sync apps: architectural fallback — move STT off streaming WS to a non-WS/batch transcribe path, OR a different transport/host. This is a DECISION, not an executor fix. Would touch bridge + backend client (transport change), gated.

## Productive work available WITHOUT STT (if Rob wants to proceed elsewhere)
- Independent small code fix (AUTO-RUN draft + show-diff): make transcribe_file/open_stream attach a message so error_message is never blank. (Does not need /stream working.)
- NOTE: TTS bridge bring-up would hit the SAME Cerebrium WS question if TTS streams over WS — verify TTS transport before committing GPU spend. Do not start TTS until the WS routing answer is known (avoids repeating this blocker on a paid GPU).

## ROB-ONLY (carried)
- Cerebrium support escalation; transport/architecture decision; any redeploy (confirm-before-live); secret/url changes; upload/reprocess; TTS/face GPU. No secret values read/set by executor.

## Hard constraints
No second redeploy / no toml/code change without show-diff + approval. No retry of 294c44ea until /stream returns non-404. No upload/reprocess/authed-call by executor. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
