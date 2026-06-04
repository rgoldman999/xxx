# CURRENT GATE

> Process governed by state/HANDOFF-POLICY.md (approval-light: AUTO-RUN / CHECKPOINT / ROB-ONLY). Read it before acting.

## Objective
Confirm R2 actually works (uploads -> r2://). R2 values corrected, backend redeployed rev 00025, health green. Need ONE fresh upload (Rob) to verify r2://. STOPPED at the upload boundary.

## Done / verified (read-only)
- STT bridge btg-stt READY; WS routing to /stream confirmed (101).
- Five R2 names present with corrected values (re-set; count 25).
- Backend redeployed rev 00025; /api/health 200.

## Next step — ROB-ONLY (the upload boundary)
1. Rob performs ONE small fresh upload (any persona) against rev 00025. (The earlier file:// uploads were on rev 00024 with bad values; they cannot become r2:// retroactively — a NEW upload is required.)
2. Then either:
   - Rob: GET {backend_base}/api/upload/_diag (authed) -> report storage_uri_val + r2_error for that event (scheme + error field, NOT secret values), OR
   - Rob gives executor the new source_id / persona; executor reads storage_uri scheme read-only from DB.
   backend_base: https://api.aws.us-east-1.cerebrium.ai/v4/p-a907d7c5/backtogether-backend
3. PASS = storage_uri starts r2:// AND r2_error null. FAIL = file:// or r2_error set -> R2 still misconfigured, diagnose; do NOT proceed.
4. On PASS (ROB-ONLY): re-upload ellis videos (or use a good source) -> reprocess.
5. (AUTO-RUN) executor verifies STT end-to-end: sources advance audio_extracted -> transcribed; source_speaker_segments > 0; btg-stt logs show /stream requests.

## Secondary (parked)
- video ad504088 ffmpeg error (rev 00024 era) — revisit after R2 confirmed if that source is reused.

## After STT validates
- TTS bridge bring-up (voice_id / callable persona). New paid GPU infra = ROB-ONLY spend approval.

## ROB-ONLY (carried)
- Upload/re-upload; authed _diag call; reprocess trigger; redeploy (confirm-before-live); TTS/face GPU. No secret values read/set by executor.

## Hard constraints
No upload/authed-call/reprocess by executor. No prod DB writes by executor. No redeploy without confirm-before-live. No blind retry of file:// ellis sources. No TTS/face.

---

## STANDING SECURITY (ROB-ONLY, not done)
Rotate CF Global API Key (uploaded to chat, full-account scope). Clear wrangler OAuth session. Active call e1f9cdd4 still active (end user-run).
