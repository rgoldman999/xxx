# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Validate upload STT end-to-end. Both apps deployed on HTTP transcribe path. The triggered reprocess did NOT land (row untouched, no job enqueued) — new path not yet exercised. Re-trigger needed.

## Done / verified
- btg-stt build-8566f671 (HTTP /api/stt/transcribe, POST smoke 401 = reachable). backend rev 00026 (/api/health 200, client httpx POSTs the route). Code 0668e69.
- REPROCESS DID NOT LAND (RESULTS 04:22Z): 294c44ea still failed/audio_extracted, updated 01:16 (unchanged); 0 sources processed since deploy; job_ledger has NO new redrive job (only old f894f233). reprocess_source resets row BEFORE enqueue; row not reset => handler didn't complete. Neither dashboard shows the _reprocess POST. => the authed POST did not reach/complete; NOT an STT result.
- Bonus: old job f894f233 last_error = TimeoutError at stt_bridge.py:220 open_stream ws.recv() — the OLD WS path our HTTP fix replaces. Confirms root-cause analysis + that the fix removes this exact call.

## Next step — ROB-ONLY: re-trigger reprocess + report response
1. Rob re-runs authed POST {backend_base}/api/upload/_reprocess/294c44ea-9784-42eb-988a-701a11d7c448 and REPORTS the HTTP status + body.
   - success body: {"status":"reset_and_enqueued","enqueued":true,"job_id":...}
   - if non-2xx (401/404/5xx): that explains the no-op; fix the call (auth/url/path) and retry.
   - backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
2. (AUTO-RUN read-only verify, after a successful enqueue): row pending -> transcribed; transcript_text populated; source_speaker_segments>0; btg-stt Runs authed POST /api/stt/transcribe 200.
3. If transcribe 200 but diarization/segments off -> diagnose (no blind retry).

## ROB-ONLY (carried)
- Re-trigger reprocess (authed); push (if wanted); Cerebrium escalation (realtime WS); TTS/face. No secret values read/set by executor.

## Hard constraints
No reprocess/authed-call by executor. No deploy. No prod DB writes by executor. No push unless asked. No TTS/face.

## Follow-on
- Realtime WS still blocked (separate). TTS/avatar WS will need same HTTP treatment at bring-up.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).