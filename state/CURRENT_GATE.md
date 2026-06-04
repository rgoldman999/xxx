# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Diagnose why the first r2:// video failed at the diarize/STT step (empty error). Get the actual exception (Rob log/inspect), then fix and re-verify. STT still not validated end-to-end.

## Done / verified
- STT bridge READY; backend rev 00025; R2 WORKING (uploads -> r2://).
- NEW video 294c44ea (persona 14578822) reprocessed: storage_uri r2://, extracted audio OK (audio_duration 2085.94s) -> then FAILED at diarize/STT, error_message EMPTY. (RESULTS 01:20Z) This is the furthest any video has gotten; the failure is now at the STT step itself.

## Blocker — STT-step failure, cause unknown read-only
- btg-stt + backend logs not retrievable (cerebrium logs flaky this session).
- Richer per-source detail is owner-authed (/_inspect). DB shows the failure but not the exception.

## Next step — ROB-ONLY: surface the actual error (pick any)
1. Rob: GET {backend_base}/api/upload/_inspect/294c44ea-9784-42eb-988a-701a11d7c448 (authed) -> report fields / error detail.
2. Rob: dashboard App Logs (backtogether-backend) ~01:15-01:16 -> the diarize/STT exception; and btg-stt App Logs -> whether a /stream request arrived.
   backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
3. Candidates to look for (NOT assumed): does the STT bridge fetch the r2:// audio itself (R2 access on the bridge?) or does the backend stream bytes to it; WS auth (bridge_auth) mismatch; transcribe_file timeout on a 35-min file; exception thrown before error_message is written.
4. Once the error is known: executor diagnoses against source (processing.py / stt_bridge.py) read-only, proposes a fix; code change = AUTO-RUN draft + show-diff, deploy = confirm-before-live. NO blind retry of 294c44ea until cause known.

## After STT validates
- TTS bridge bring-up (voice_id / callable persona). New paid GPU infra = ROB-ONLY spend.

## ROB-ONLY (carried)
- Authed _inspect call / dashboard logs; upload/reprocess; redeploy (confirm-before-live); TTS/face. No secret values read/set by executor.

## Hard constraints
No authed-call/upload/reprocess by executor. No prod DB writes by executor. No blind retry of 294c44ea or the file:// ellis sources. No redeploy without confirm-before-live. No TTS/face.

## Note
Console error Rob saw ("listener indicated async response... message channel closed") = browser-extension message, unrelated to backend/STT.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
