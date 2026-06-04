# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Resolve the CONFIRMED STT root cause: backend WS connect to /stream returns 404 at the Cerebrium v4 gateway (WS upgrade not proxied). Determine whether Cerebrium v4 supports WebSockets before any fix. STT not validated end-to-end.

## Root cause — CONFIRMED (RESULTS 01:37Z, dashboard Runs + source)
- btg-stt Runs: /stream GET -> 404 (18:15:30, the 294c44ea reprocess; and 15:53:25). /healthz GET -> 200. Same app.
- Backend does a real WS upgrade (websockets.connect wss://host/stream, stt_bridge.py:211). Bridge registers @router.websocket("/stream") (stt.py:205, no prefix). Dashboard logs inbound as "/stream GET" (plain GET) -> FastAPI ws route doesn't match a GET -> 404.
- => Cerebrium v4 sync endpoint is NOT proxying the WS Upgrade to the container. WS handshake never reaches the bridge -> 404 -> messageless exception -> processing.py:407 sets blank error -> fail at audio_extracted. All symptoms explained. (Earlier direct 101 probe bypassed the gateway.)

## Next step — read-only research (AUTO-RUN) + DECISION (Rob)
1. (AUTO-RUN) executor web-searches Cerebrium docs: does a v4 app support WebSocket-upgrade proxying? Is wss to api.aws...v4/p-.../btg-stt/stream supported, or is there a dedicated WS endpoint/host/path/invocation?
2. Outcomes:
   a. WS supported, different URL shape -> fix _ws_url_from / STT_SERVICE_URL base (code is AUTO-RUN draft+show-diff; any secret/url value change is ROB-ONLY; deploy is confirm-before-live).
   b. v4 does NOT proxy WS -> ARCHITECTURAL DECISION (Rob/Jeannine): the bridge's streaming /stream WS design may be incompatible with Cerebrium v4 request/response apps. Options would include a non-WS/batch transcribe path, or a different transport/host for STT. NOT an executor fix — escalate.
3. HALT before any code change/redeploy/retry until the WS-support question is answered.

## ROB-ONLY (carried)
- Architectural decision if WS unsupported; any secret or url value change; redeploy (confirm-before-live); upload/reprocess; TTS/face. No secret values read or set by executor.

## Hard constraints
No code change/deploy/retry until Cerebrium WS support is known. No executor upload/authed-call/reprocess. No prod DB writes by executor. No TTS/face.

## Note
- Independent small fix worth doing later: make transcribe_file/open_stream attach a message (type name) so the failure's error string is never blank.
- Console error Rob saw = browser-extension noise, unrelated.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
