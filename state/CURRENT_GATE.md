# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Confirm the exact STT-step exception for source 294c44ea, then fix and re-verify STT end-to-end. Diagnosis (read-only) done; leading hypothesis = WS timeout/cold-start on a 35-min file. Needs the exception TYPE from logs to confirm.

## Done / verified
- STT bridge READY; backend rev 00025; R2 WORKING.
- 294c44ea: r2:// source, audio extracted (2085s), failed at diarize/STT, error_message EMPTY.
- Diagnosis (RESULTS 01:26Z): two red herrings ELIMINATED in source —
  (1) env_stt_bridge_url_present=false is misleading: _inspect checks legacy STT_BRIDGE_URL/BRIDGE_URL; the client reads stt_service_url=STT_SERVICE_URL (set+verified). Bridge URL IS configured.
  (2) r2:// audio is NOT passed to STT; processing.py passes a LOCAL .wav. transcribe_file got a real local file.
- Empty error explained: handler sets error_message=str(e)[:500]; str(e) is empty -> messageless exception (timeout/cancel/connection-close).
- LEADING HYPOTHESIS: STT WS call failed messagelessly; btg-stt is scale-to-zero, so reprocess WS connect likely raced GPU cold-start (open_timeout ~15s vs container spin-up + whisper load ~9s), OR long-file (35-min) timeout. NOT confirmed (exception type only in logs).

## Next step — ROB-ONLY: confirm the exception type
1. Rob: dashboard btg-stt App Logs AND backtogether-backend App Logs ~01:15-01:16 -> the exact exception at the WS call (timeout / connection refused / cold-start / mid-stream disconnect). Report the exception type + message.
   (cerebrium CLI logs are flaky/empty this session, so dashboard logs are the reliable source.)

## Then fix (per confirmed cause) — executor drafts, Rob approves deploy
- If cold-start race: pre-warm btg-stt (min_replicas=1 temporarily = ROB-ONLY spend change) for the test, OR add WS connect retry / raise open_timeout in stt_bridge.py (AUTO-RUN draft + show-diff; confirm-before-live deploy).
- If long-file timeout: raise per-file timeout or chunk audio (same gating).
- Independent small fix: make transcribe_file attach a message (type name) so error_message is never blank (AUTO-RUN draft + show-diff).

## After STT validates
- TTS bridge bring-up. New paid GPU infra = ROB-ONLY spend.

## ROB-ONLY (carried)
- Dashboard logs; min_replicas/spend change; upload/reprocess; redeploy (confirm-before-live); TTS/face. No secret values read/set by executor.

## Hard constraints
No retry of 294c44ea until exception type known. No executor upload/authed-call/reprocess/deploy. No prod DB writes by executor. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
